Build an On-Premises AI Kaggle Competition Assistant with Qdrant RAG and Ollama

https://n8nworkflows.xyz/workflows/build-an-on-premises-ai-kaggle-competition-assistant-with-qdrant-rag-and-ollama-3967


# Build an On-Premises AI Kaggle Competition Assistant with Qdrant RAG and Ollama

---

### 1. Workflow Overview

This workflow implements an **On-Premises AI Kaggle Competition Assistant** that leverages Large Language Models (LLMs), Retrieval-Augmented Generation (RAG), and vector databases to assist users in Kaggle competitions, specifically tested on disaster-tweet classification tasks. It combines file system monitoring, document ingestion and summarization, vector embedding for semantic search, and an interactive LLM chat interface for coding assistance.

The workflow is structured into three main logical blocks:

- **1.1 Document Ingestion and Processing**  
  Watches a local folder for new files (e.g., Jupyter notebooks), imports and extracts their content, converts HTML if necessary, summarizes the documents, and indexes them into a Qdrant vector store for retrieval.

- **1.2 Vector Store Management and Embedding**  
  Generates vector embeddings for documents using Ollama embedding models, and inserts these vectors into the Qdrant vector database to enable retrieval-augmented queries.

- **1.3 Interactive AI Chat Agent with Retrieval Tools**  
  Listens for chat messages, invokes an AI Agent configured as a Kaggle data science assistant, integrates retrieval from the vector store to provide context-aware multi-turn conversations, including real-time coding advice and debugging.

This modular design supports privacy by running fully on-premises with GPU-accelerated Ollama models and tightly integrates n8n orchestration with AI services.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion and Processing

**Overview:**  
This block monitors a local folder for new files, imports and extracts their content, handles file type differentiation (e.g., HTML vs. text), converts HTML to markdown, summarizes the content, and prepares data for vector embedding.

**Nodes Involved:**  
- Local File Trigger  
- Settings  
- Import File  
- Get FileType (Switch)  
- Extract from TEXT  
- Markdown  
- Ollama Summarizer  
- Summarization Chain  
- Merge  

**Node Details:**

- **Local File Trigger**  
  - *Type:* Local file system event trigger  
  - *Purpose:* Watches folder `C:\\ipynb\\loadme` for newly added files using polling and awaiting write finish to ensure file completeness.  
  - *Inputs:* None (trigger node)  
  - *Outputs:* Emits file path data to "Settings" node.  
  - *Failure Modes:* File access permission errors, incomplete writes, polling delays.

- **Settings**  
  - *Type:* Set node  
  - *Purpose:* Extracts and sets file path and filename from trigger data for downstream processing.  
  - *Key Variables:* `path` = fixed folder path, `filename` = last segment of the path string.  
  - *Inputs:* From Local File Trigger  
  - *Outputs:* To Import File node.

- **Import File**  
  - *Type:* Read/Write File node  
  - *Purpose:* Reads the full content of the file located at the combined path + filename.  
  - *Inputs:* From Settings node  
  - *Outputs:* To Get FileType node.  
  - *Failures:* File not found if deleted between trigger and read, encoding issues.

- **Get FileType**  
  - *Type:* Switch  
  - *Purpose:* Determines if the file type is "html" to branch processing accordingly (currently configured only for "html" check).  
  - *Inputs:* From Import File  
  - *Outputs:* To Extract from TEXT (default branch)  
  - *Failure:* Unhandled file types.

- **Extract from TEXT**  
  - *Type:* Extract from file  
  - *Purpose:* Extracts text content from imported file (assumed non-html).  
  - *Inputs:* From Get FileType  
  - *Outputs:* To Markdown node.

- **Markdown**  
  - *Type:* Conversion node  
  - *Purpose:* Converts extracted HTML content (from file) to markdown format for easier summarization.  
  - *Inputs:* From Extract from TEXT  
  - *Outputs:* To Ollama Summarizer.

- **Ollama Summarizer**  
  - *Type:* Language Model (LLM) summarization node  
  - *Model:* `ALIENTELLIGENCE/contentsummarizer:latest`  
  - *Purpose:* Summarizes document content with refine method for better accuracy.  
  - *Inputs:* From Markdown  
  - *Outputs:* To Summarization Chain.

- **Summarization Chain**  
  - *Type:* Langchain summarization chain  
  - *Purpose:* Further refines and finalizes summary output, chunking large texts into 4000-token chunks.  
  - *Inputs:* From Ollama Summarizer  
  - *Outputs:* To Merge node.

- **Merge**  
  - *Type:* Merge node  
  - *Purpose:* Combines data streams for downstream vector embedding.  
  - *Inputs:* From Summarization Chain (main), and from Qdrant Vector Store (insert confirmation)  
  - *Outputs:* None further in this block.

---

#### 2.2 Vector Store Management and Embedding

**Overview:**  
This block generates embeddings from summarized documents and inserts these embeddings into the Qdrant vector store to enable semantic document retrieval for RAG.

**Nodes Involved:**  
- Embeddings Ollama  
- Qdrant Vector Store  

**Node Details:**

- **Embeddings Ollama**  
  - *Type:* Embeddings generation node using Ollama API  
  - *Model:* `mxbai-embed-large:latest`  
  - *Purpose:* Transforms summarized document text into vector embeddings compatible with Qdrant.  
  - *Inputs:* From Merge node (merged summarized content)  
  - *Outputs:* To Qdrant Vector Store (insert mode)  
  - *Failures:* Ollama API connectivity, GPU resource exhaustion.

- **Qdrant Vector Store**  
  - *Type:* Vector database node (Qdrant)  
  - *Purpose:* Inserts embeddings into the `test_rag` collection for later retrieval.  
  - *Credentials:* Requires configured Qdrant API credentials.  
  - *Inputs:* From Embeddings Ollama  
  - *Outputs:* To Merge node for confirmation (loopback).

---

#### 2.3 Interactive AI Chat Agent with Retrieval Tools

**Overview:**  
This block handles incoming user chat messages, processes them through an AI agent specialized in Kaggle data science tasks, integrates vector store retrieval to provide relevant background context, maintains conversational memory, and outputs AI-generated responses.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- Vector Store Tool  
- Window Buffer Memory  
- Qdrant Vector Store2  
- Embeddings Ollama2  
- Ollama Chat Model3  
- Ollama Chat Model4  

**Node Details:**

- **When chat message received**  
  - *Type:* Langchain Chat Trigger (webhook)  
  - *Purpose:* Entry point for user chat messages, triggered by webhook ID.  
  - *Inputs:* External user messages  
  - *Outputs:* To AI Agent.

- **AI Agent**  
  - *Type:* Langchain Agent node  
  - *Purpose:* Core LLM agent configured with a system message instructing it to act as a Kaggle Python programming assistant, including multi-turn debugging and iterative code refinement.  
  - *System Message:* Detailed instructions on research, use of the "previous_entry" tool, question asking, and debugging.  
  - *Inputs:* From chat trigger, and from Vector Store Tool & Window Buffer Memory (for retrieval and memory)  
  - *Outputs:* Final AI responses to chat.

- **Vector Store Tool**  
  - *Type:* Langchain Tool node  
  - *Purpose:* Provides the AI Agent a retrieval tool named "previous_entry" which queries the Qdrant vector store for relevant documents based on chat input.  
  - *Description:* Dynamically set to the current chat input to fetch relevant context.  
  - *Inputs:* From Ollama Chat Model4  
  - *Outputs:* To AI Agent.

- **Window Buffer Memory**  
  - *Type:* Memory node  
  - *Purpose:* Maintains a sliding window of last 15 conversational exchanges to preserve context in multi-turn dialogue.  
  - *Inputs:* From AI Agent  
  - *Outputs:* To AI Agent (feedback loop).

- **Qdrant Vector Store2**  
  - *Type:* Vector store query node  
  - *Purpose:* Searches `test_rag` collection to retrieve documents relevant to the current chat query for the Vector Store Tool.  
  - *Credentials:* Same Qdrant API credentials as earlier node.  
  - *Inputs:* From Embeddings Ollama2 (query embeddings)  
  - *Outputs:* To Vector Store Tool.

- **Embeddings Ollama2**  
  - *Type:* Embeddings generator for chat queries  
  - *Model:* `mxbai-embed-large:latest`  
  - *Purpose:* Converts chat inputs to vectors for similarity search in Qdrant.  
  - *Inputs:* From Vector Store Tool (or chat input)  
  - *Outputs:* To Qdrant Vector Store2.

- **Ollama Chat Model3**  
  - *Type:* Chat LLM model node  
  - *Model:* `qwen3:8b`  
  - *Purpose:* Provides LLM predictions to the AI Agent.  
  - *Inputs:* From When chat message received  
  - *Outputs:* To AI Agent.

- **Ollama Chat Model4**  
  - *Type:* Chat LLM model node  
  - *Model:* `qwen3:8b`  
  - *Purpose:* Supports the Vector Store Tool with chat model capabilities.  
  - *Inputs:* From Qdrant Vector Store2  
  - *Outputs:* To Vector Store Tool.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                          | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                                     |
|---------------------------|----------------------------------|----------------------------------------|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Local File Trigger         | n8n-nodes-base.localFileTrigger  | Watches folder for new files            | None                         | Settings                      | ## Step 1. Watch Folder and Import New Documents [Read more about Local File Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.localfiletrigger) With n8n's local file trigger, we're able to trigger the workflow when files are created in our target folder. We still have to import them however as the trigger will only give the file's path. The "Extract From" node is used to get at the file's contents. |
| Settings                  | n8n-nodes-base.set               | Sets path and filename variables        | Local File Trigger           | Import File                   | (Same as above)                                                                                                                 |
| Import File                | n8n-nodes-base.readWriteFile     | Reads file content                      | Settings                     | Get FileType                  | (Same as above)                                                                                                                 |
| Get FileType               | n8n-nodes-base.switch            | Determines file type (html vs others)  | Import File                  | Extract from TEXT             | (Same as above)                                                                                                                 |
| Extract from TEXT          | n8n-nodes-base.extractFromFile   | Extracts text content from file         | Get FileType                 | Markdown                     | (Same as above)                                                                                                                 |
| Markdown                   | n8n-nodes-base.markdown          | Converts HTML to markdown                | Extract from TEXT            | Ollama Summarizer            | (Same as above)                                                                                                                 |
| Ollama Summarizer          | @n8n/n8n-nodes-langchain.lmOllama| Summarizes document content             | Markdown                     | Summarization Chain          | ## Step 2. Summarise and Vectorise Document Contents [Learn more about using the Qdrant VectorStore](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreqdrant) Capturing the document into our vector store is intended for a technique we'll use later known as Retrieval Augumented Generation or "RAG" for short. For our scenario, this allows our LLM to retrieve context more efficiently which produces better respsonses. |
| Summarization Chain       | @n8n/n8n-nodes-langchain.chainSummarization | Refines and finalizes summaries        | Ollama Summarizer            | Merge                        | (Same as above)                                                                                                                 |
| Merge                     | n8n-nodes-base.merge             | Merges data streams                     | Summarization Chain, Qdrant Vector Store | Embeddings Ollama (via connection) | (Same as above)                                                                                                                 |
| Embeddings Ollama          | @n8n/n8n-nodes-langchain.embeddingsOllama | Generates vector embeddings             | Merge                        | Qdrant Vector Store          | (Same as above)                                                                                                                 |
| Qdrant Vector Store        | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Inserts embeddings into Qdrant          | Embeddings Ollama            | Merge                        | (Same as above)                                                                                                                 |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Triggers workflow on chat input         | External webhook             | AI Agent                     |                                                                                                                                |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent  | Core Kaggle data science AI assistant   | When chat message received, Vector Store Tool, Window Buffer Memory | Window Buffer Memory, Final output |                                                                                                                                |
| Vector Store Tool          | @n8n/n8n-nodes-langchain.toolVectorStore | Retrieval tool for AI Agent              | Ollama Chat Model4           | AI Agent                     |                                                                                                                                |
| Window Buffer Memory       | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversational context       | AI Agent                     | AI Agent                     |                                                                                                                                |
| Qdrant Vector Store2       | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Queries Qdrant for relevant documents   | Embeddings Ollama2           | Ollama Chat Model4           |                                                                                                                                |
| Embeddings Ollama2         | @n8n/n8n-nodes-langchain.embeddingsOllama | Embeddings for chat query vectors       | Vector Store Tool            | Qdrant Vector Store2         |                                                                                                                                |
| Ollama Chat Model3         | @n8n/n8n-nodes-langchain.lmChatOllama | Provides chat LLM for AI Agent           | When chat message received   | AI Agent                     |                                                                                                                                |
| Ollama Chat Model4         | @n8n/n8n-nodes-langchain.lmChatOllama | Chat LLM for Vector Store Tool           | Qdrant Vector Store2         | Vector Store Tool            |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Local File Trigger" node**  
   - Type: Local File Trigger  
   - Parameters:  
     - Path: `C:\\ipynb\\loadme`  
     - Events: `add`  
     - Options: Enable `usePolling`, `followSymlinks`, `awaitWriteFinish`  
   - Position: Place at top-left area.

2. **Create "Settings" node**  
   - Type: Set  
   - Parameters:  
     - Set `path` as string: `C:\\ipynb\\loadme\\`  
     - Set `filename` as expression: `={{ $json.path.split('\\').pop() }}`  
   - Connect: Output from Local File Trigger → Input to Settings.

3. **Create "Import File" node**  
   - Type: Read/Write File  
   - Parameters:  
     - File Selector: `={{ $json.path }}{{ $json.filename }}`  
   - Connect: Settings → Import File.

4. **Create "Get FileType" node**  
   - Type: Switch  
   - Parameters:  
     - Rule: Check if `$json.fileType` equals `html` (case sensitive)  
   - Connect: Import File → Get FileType.

5. **Create "Extract from TEXT" node**  
   - Type: Extract from File  
   - Parameters:  
     - Operation: `text`  
   - Connect: Get FileType → Extract from TEXT (default branch).

6. **Create "Markdown" node**  
   - Type: Markdown  
   - Parameters:  
     - HTML input: `={{ $json.data }}`  
   - Connect: Extract from TEXT → Markdown.

7. **Create "Ollama Summarizer" node**  
   - Type: Langchain LLM Ollama  
   - Credentials: Configure Ollama API credentials  
   - Parameters:  
     - Model: `ALIENTELLIGENCE/contentsummarizer:latest`  
   - Connect: Markdown → Ollama Summarizer.

8. **Create "Summarization Chain" node**  
   - Type: Langchain Summarization Chain  
   - Parameters:  
     - Summarization method: `refine`  
     - Chunk Size: `4000` tokens  
   - Connect: Ollama Summarizer → Summarization Chain.

9. **Create "Merge" node**  
   - Type: Merge  
   - Connect: Summarization Chain → Merge.

10. **Create "Embeddings Ollama" node**  
    - Type: Langchain Embeddings Ollama  
    - Credentials: Same Ollama API credentials  
    - Parameters:  
      - Model: `mxbai-embed-large:latest`  
    - Connect: Merge → Embeddings Ollama.

11. **Create "Qdrant Vector Store" node**  
    - Type: Langchain Vector Store Qdrant  
    - Credentials: Configure Qdrant API credentials  
    - Parameters:  
      - Mode: `insert`  
      - Collection: `test_rag`  
    - Connect: Embeddings Ollama → Qdrant Vector Store  
    - Connect Qdrant Vector Store output → Merge (to close loop).

12. **Create "When chat message received" node**  
    - Type: Langchain Chat Trigger  
    - Parameters:  
      - Webhook ID: auto-generated or specified  
    - Connect: None; entry point.

13. **Create "AI Agent" node**  
    - Type: Langchain Agent  
    - Parameters:  
      - System Message:  
        ```
        This is a helpful and exacting data science LLM model and master Kaggle python programmer.

        If Kaggle contest requirements are given from the chat input; first deeply research the problem.

        Access the tool: "previous_entry" when preparing your background research.

        Then Ask any needed questions to clarify and understand the requirements necessary to build a program to address the challenge.

        Review your proposed program for errors and bugs.

        Then present the program.

        If errors are returned; then iteratively debug with the chat user.
        ```  
    - Connect: When chat message received → AI Agent.

14. **Create "Window Buffer Memory" node**  
    - Type: Langchain Memory Buffer Window  
    - Parameters:  
      - Context Window Length: `15`  
    - Connect: AI Agent → Window Buffer Memory → AI Agent (feedback loop).

15. **Create "Vector Store Tool" node**  
    - Type: Langchain Tool Vector Store  
    - Parameters:  
      - Name: `previous_entry`  
      - Description: `={{ $('When chat message received').item.json.chatInput }}`  
    - Connect: Ollama Chat Model4 → Vector Store Tool → AI Agent.

16. **Create "Qdrant Vector Store2" node**  
    - Type: Langchain Vector Store Qdrant  
    - Credentials: Same Qdrant API credentials  
    - Parameters:  
      - Collection: `test_rag`  
      - Mode: Query (default)  
    - Connect: Embeddings Ollama2 → Qdrant Vector Store2 → Ollama Chat Model4.

17. **Create "Embeddings Ollama2" node**  
    - Type: Langchain Embeddings Ollama  
    - Credentials: Ollama API credentials  
    - Parameters:  
      - Model: `mxbai-embed-large:latest`  
    - Connect: Vector Store Tool → Embeddings Ollama2 → Qdrant Vector Store2.

18. **Create "Ollama Chat Model3" node**  
    - Type: Langchain Chat Ollama  
    - Credentials: Ollama API credentials  
    - Parameters:  
      - Model: `qwen3:8b`  
    - Connect: When chat message received → Ollama Chat Model3 → AI Agent.

19. **Create "Ollama Chat Model4" node**  
    - Type: Langchain Chat Ollama  
    - Credentials: Ollama API credentials  
    - Parameters:  
      - Model: `qwen3:8b`  
    - Connect: Qdrant Vector Store2 → Ollama Chat Model4 → Vector Store Tool.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow tested on binary disaster-tweet classification Kaggle competition dataset.                                                                                                                                             | Workflow Description                                                                                           |
| GPU acceleration is required for smooth operation due to local Ollama LLM models and embedding generation demands.                                                                                                              | Workflow Description                                                                                           |
| The workflow is designed for fully on-premises deployment to ensure data privacy and no external code or data transfer occurs.                                                                                                  | Workflow Description                                                                                           |
| Step 1 Sticky Note: Explains local file trigger usage and import process.                                                                                                                                                        | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.localfiletrigger                            |
| Step 2 Sticky Note: Explains summarization and vectorization using Qdrant for RAG technique.                                                                                                                                     | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreqdrant        |
| Based on prior workflow #2339 which breaks down documents into study notes using templating and Qdrant.                                                                                                                         | https://n8n.io/workflows/2339                                                                                   |
| Ollama models used: `ALIENTELLIGENCE/contentsummarizer:latest` for summarization, `qwen3:8b` for chat and coding, and `mxbai-embed-large:latest` for embeddings.                                                                | Workflow Description                                                                                           |
| The AI Agent’s system message configures it as a Kaggle-master Python programmer with iterative debugging and multi-turn conversation capabilities.                                                                             | AI Agent node details                                                                                           |
| Retrieval-augmented generation is implemented by querying Qdrant vector store with embeddings generated from chat inputs, enabling the AI Agent to access relevant documents dynamically during chat.                            | Vector Store Tool and related nodes                                                                             |

---

**Disclaimer:** The text above originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.