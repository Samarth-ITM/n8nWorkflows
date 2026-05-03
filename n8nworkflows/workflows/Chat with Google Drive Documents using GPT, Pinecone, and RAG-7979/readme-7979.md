Chat with Google Drive Documents using GPT, Pinecone, and RAG

https://n8nworkflows.xyz/workflows/chat-with-google-drive-documents-using-gpt--pinecone--and-rag-7979


# Chat with Google Drive Documents using GPT, Pinecone, and RAG

### 1. Workflow Overview

This workflow is designed to enable conversational interaction ("chat") with documents stored in a specified Google Drive folder, leveraging GPT-based large language models (LLMs), Pinecone vector database for retrieval, and Retrieval-Augmented Generation (RAG) techniques. It automates the ingestion of newly created or updated Google Drive files into a Pinecone vector store as embeddings and supports querying this knowledge base via a chat interface enhanced with memory to maintain context.

The workflow logically divides into the following functional blocks:

- **1.1 Google Drive Document Ingestion**: Monitor a specific Google Drive folder for new or updated files, download these files, split their content for processing, generate embeddings, and insert them into a Pinecone vector store.

- **1.2 Vector Store Construction and Management**: Create and maintain the vector store using Pinecone, connected to OpenAI embeddings for indexing the document content.

- **1.3 Conversational AI Agent**: Receive chat messages, utilize memory buffers to keep conversational context, query the vector store via a retrieval tool, and generate AI responses using GPT-4o.

- **1.4 Embeddings and Text Splitting Utilities**: Tools for creating embeddings and splitting text documents to prepare data for indexing.

- **1.5 Supporting Nodes and Metadata**: Sticky notes for documentation and UI clarity.

---

### 2. Block-by-Block Analysis

#### 2.1 Google Drive Document Ingestion

**Overview:**  
This block watches a specified Google Drive folder for file creation or updates. Upon such events, it downloads the affected file(s), processes their content by splitting into chunks, generates embeddings, and inserts these into the Pinecone vector store. This keeps the knowledge base up-to-date automatically.

**Nodes Involved:**  
- Google Drive File Created  
- Google Drive File Updated  
- Download File From Google Drive  
- Recursive Character Text Splitter  
- Default Data Loader  
- Pinecone Vector Store  

**Node Details:**

- **Google Drive File Created**  
  - *Type:* Trigger node (Google Drive Trigger)  
  - *Role:* Watches for new files created in a specific Drive folder ("4. Personal Helper")  
  - *Config:* Triggers every minute, filters by folder ID  
  - *Connections:* Outputs to "Download File From Google Drive"  
  - *Failure cases:* Auth errors, folder permission issues, API rate limits

- **Google Drive File Updated**  
  - *Type:* Trigger node (Google Drive Trigger)  
  - *Role:* Watches for updated files in the same folder as above  
  - *Config:* Similar to "File Created," triggers every minute  
  - *Connections:* Outputs to "Download File From Google Drive"  
  - *Failure cases:* Same as above

- **Download File From Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the file identified by the trigger event  
  - *Config:* Uses file ID and file name from previous node’s JSON data  
  - *Credentials:* Google Drive OAuth2  
  - *Connections:* Outputs to "Pinecone Vector Store"  
  - *Failure cases:* File not accessible, download errors, expired tokens

- **Recursive Character Text Splitter**  
  - *Type:* Text splitter  
  - *Role:* Splits large document text into chunks for embedding generation  
  - *Config:* Chunk size: 2000 characters, overlap: 100 characters  
  - *Connections:* Input from "Default Data Loader", output to "Default Data Loader"  
  - *Failure cases:* Incorrect input format, empty documents

- **Default Data Loader**  
  - *Type:* Document loader  
  - *Role:* Loads and prepares document data for embedding insertion  
  - *Connections:* Input from "Recursive Character Text Splitter", output to "Pinecone Vector Store"  
  - *Failure cases:* Unsupported document formats, parsing errors

- **Pinecone Vector Store**  
  - *Type:* Vector store insertion node  
  - *Role:* Inserts document embeddings into the Pinecone index ("my-docs")  
  - *Credentials:* Pinecone API credentials  
  - *Connections:* Inputs from "Default Data Loader" and "Download File From Google Drive"  
  - *Failure cases:* API errors, quota exceeded, network issues

---

#### 2.2 Vector Store Construction and Management

**Overview:**  
This block manages the vector store that indexes document embeddings used for retrieval during chat interactions. It leverages Pinecone as the vector database and OpenAI to generate embeddings.

**Nodes Involved:**  
- Embeddings OpenAI1  
- Pinecone Vector Store (Retrieval)  
- Vector Store Tool  

**Node Details:**

- **Embeddings OpenAI1**  
  - *Type:* Embedding generator  
  - *Role:* Generates embeddings from document text using OpenAI's "text-embedding-3-small" model  
  - *Credentials:* OpenAI API  
  - *Connections:* Output feeds "Pinecone Vector Store" for insertion  
  - *Failure cases:* API quota limits, input format errors

- **Pinecone Vector Store (Retrieval)**  
  - *Type:* Vector store retrieval node  
  - *Role:* Queries the Pinecone index to retrieve relevant documents based on a query embedding  
  - *Credentials:* Pinecone API  
  - *Connections:* Output connects to "Vector Store Tool"  
  - *Failure cases:* Query failures, index not found, connectivity issues

- **Vector Store Tool**  
  - *Type:* Tool node (Langchain)  
  - *Role:* Wraps the vector store retrieval to be used as a tool by the AI agent for document search  
  - *Connections:* Input from "Pinecone Vector Store (Retrieval)" and output to "AI Sales Agent"  
  - *Failure cases:* Tool initialization failures

---

#### 2.3 Conversational AI Agent

**Overview:**  
Handles user chat input, maintains conversation memory, queries the vector store tool for relevant documents, and generates responses using the OpenAI GPT-4o model.

**Nodes Involved:**  
- When chat message received  
- AI Sales Agent  
- Window Buffer Memory  
- OpenAI Chat Model  
- OpenAI Chat Model1  

**Node Details:**

- **When chat message received**  
  - *Type:* Chat trigger (Langchain)  
  - *Role:* Public webhook that receives chat input messages from users  
  - *Config:* Public access, no special options  
  - *Connections:* Outputs to "AI Sales Agent"  
  - *Failure cases:* Webhook downtime, improper payload format

- **AI Sales Agent**  
  - *Type:* Agent node (Langchain)  
  - *Role:* Core conversational AI agent that processes chat input, uses memory, tools, and language models to respond  
  - *Config:* System message defines assistant as a personal assistant focused on factual answers about internal company PDFs; responses limited to 100-200 words; short, human-like answers  
  - *Connections:* Inputs from chat trigger, memory, tool, and language model nodes; outputs response to webhook  
  - *Failure cases:* Model API errors, memory key conflicts, incomplete tool availability

- **Window Buffer Memory**  
  - *Type:* Memory node (Langchain)  
  - *Role:* Maintains recent conversation messages per user session to provide context for the agent  
  - *Config:* Session key based on user identifier from chat input  
  - *Connections:* Linked as memory input to "AI Sales Agent"  
  - *Failure cases:* Session key resolution errors, memory overflow (unlikely)

- **OpenAI Chat Model**  
  - *Type:* Language model node (Langchain)  
  - *Role:* Provides GPT-4o language generation for the conversational agent  
  - *Config:* Model set to "gpt-4o-2024-08-06" with default options  
  - *Connections:* Connected as language model input to the "AI Sales Agent"  
  - *Failure cases:* API rate limits, auth errors, model unavailability

- **OpenAI Chat Model1**  
  - *Type:* Language model node (Langchain)  
  - *Role:* Used by vector store tool to generate language-based queries/responses  
  - *Config:* Same GPT-4o model as above  
  - *Connections:* Connected as language model input to "Vector Store Tool"  
  - *Failure cases:* Same as above

---

#### 2.4 Embeddings and Text Splitting Utilities

**Overview:**  
These nodes provide embedding generation and text chunking functionality essential for both document ingestion and query processing.

**Nodes Involved:**  
- Embeddings OpenAI  
- Embeddings OpenAI1 (already covered in 2.2)  
- Recursive Character Text Splitter (already covered in 2.1)  

**Node Details:**

- **Embeddings OpenAI**  
  - *Type:* Embeddings node (Langchain)  
  - *Role:* Generates text embeddings for search queries or document content  
  - *Config:* Uses "text-embedding-3-small" model  
  - *Credentials:* OpenAI API  
  - *Connections:* Connected to "Pinecone Vector Store (Retrieval)" for query embeddings  
  - *Failure cases:* API rate limits, malformed text input

---

#### 2.5 Supporting Nodes and Metadata

**Overview:**  
Sticky note nodes provide in-workflow documentation and links to relevant n8n documentation for users or developers.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note3  
- Sticky Note5  

**Node Details:**

- **Sticky Note**  
  - Content: "## 1. Build Database" with link to HTTP Request tool documentation  
  - Positioned near data ingestion nodes  

- **Sticky Note1**  
  - Content: "## 2. Create a Vector Store" with Pinecone in-memory vector store documentation link  

- **Sticky Note3**  
  - Content: "## Add documents to vector store when updating or creating new documents in Google Drive"  
  - Highlights the document ingestion trigger nodes

- **Sticky Note5**  
  - Content: "## 4. AI Agent Responds" with link to AI agent documentation; explains conversational AI agent capabilities  

These notes provide contextual guidance and learning resources directly inside the workflow editor.

---

### 3. Summary Table

| Node Name                      | Node Type                                     | Functional Role                          | Input Node(s)                           | Output Node(s)                    | Sticky Note                                                                                                  |
|--------------------------------|-----------------------------------------------|----------------------------------------|----------------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Google Drive File Created       | Google Drive Trigger                           | Trigger on new files in Drive folder   | -                                      | Download File From Google Drive   | ## Add documents to vector store when updating or creating new documents in Google Drive                     |
| Google Drive File Updated       | Google Drive Trigger                           | Trigger on updated files in Drive      | -                                      | Download File From Google Drive   | ## Add documents to vector store when updating or creating new documents in Google Drive                     |
| Download File From Google Drive | Google Drive node                              | Downloads files for processing         | Google Drive File Created, File Updated | Pinecone Vector Store             |                                                                                                              |
| Recursive Character Text Splitter | Text Splitter (Langchain)                     | Splits text into chunks for embedding  | Default Data Loader                     | Default Data Loader               |                                                                                                              |
| Default Data Loader             | Document Loader (Langchain)                    | Loads document text for processing     | Recursive Character Text Splitter      | Pinecone Vector Store             |                                                                                                              |
| Pinecone Vector Store           | Vector Store (Langchain)                       | Inserts embeddings into Pinecone index | Download File From Google Drive, Default Data Loader | -                                |                                                                                                              |
| Embeddings OpenAI              | Embeddings Generator (Langchain)               | Generates embeddings from text         | -                                      | Pinecone Vector Store (Retrieval) |                                                                                                              |
| Embeddings OpenAI1             | Embeddings Generator (Langchain)               | Generates embeddings for indexing      | Default Data Loader                     | Pinecone Vector Store             |                                                                                                              |
| Pinecone Vector Store (Retrieval) | Vector Store (Langchain)                      | Retrieves relevant documents            | Embeddings OpenAI                      | Vector Store Tool                |                                                                                                              |
| Vector Store Tool              | Tool (Langchain)                               | Provides document search functionality | Pinecone Vector Store (Retrieval)      | AI Sales Agent                   |                                                                                                              |
| When chat message received      | Chat Trigger (Langchain)                        | Receives user chat input                | -                                      | AI Sales Agent                   |                                                                                                              |
| AI Sales Agent                 | Agent (Langchain)                              | Conversational AI agent                 | When chat message received, Window Buffer Memory, Vector Store Tool, OpenAI Chat Model | -                                | ## 4. AI Agent Responds                                                                                        |
| Window Buffer Memory           | Memory Buffer (Langchain)                      | Maintains conversation context         | -                                      | AI Sales Agent                   |                                                                                                              |
| OpenAI Chat Model              | Language Model (Langchain)                      | Generates chat responses                | -                                      | AI Sales Agent                   |                                                                                                              |
| OpenAI Chat Model1             | Language Model (Langchain)                      | Supports vector store tool queries     | -                                      | Vector Store Tool                |                                                                                                              |
| Default Data Loader            | Document Loader (Langchain)                     | Loads documents for vector store       | Recursive Character Text Splitter      | Pinecone Vector Store            |                                                                                                              |
| Sticky Note                   | Sticky Note                                    | Documentation                          | -                                      | -                                | ## 1. Build Database                                                                                           |
| Sticky Note1                  | Sticky Note                                    | Documentation                          | -                                      | -                                | ## 2. Create a Vector Store                                                                                   |
| Sticky Note3                  | Sticky Note                                    | Documentation                          | -                                      | -                                | ## Add documents to vector store when updating or creating new documents in Google Drive                      |
| Sticky Note5                  | Sticky Note                                    | Documentation                          | -                                      | -                                | ## 4. AI Agent Responds                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Triggers**  
   - Add two Google Drive Trigger nodes: "Google Drive File Created" and "Google Drive File Updated".  
   - Configure both to watch the specific folder by its ID (folder: "1WlejjfGrijAhMsJ2zWUa-2OZsiXIj4Bu").  
   - Set polling to every minute.  
   - Set credentials with a valid Google Drive OAuth2 account.

2. **Download Files from Google Drive**  
   - Add a Google Drive node named "Download File From Google Drive".  
   - Configure to download files using the file ID from the trigger node JSON data (`{{$json.id}}`).  
   - Connect outputs from both Drive Trigger nodes to this node.  
   - Use the same Google Drive OAuth2 credentials.

3. **Text Splitting and Document Loading**  
   - Add a "Recursive Character Text Splitter" node.  
   - Set chunk size to 2000 characters, chunk overlap to 100 characters.  
   - Connect its output to a "Default Data Loader" node.  
   - This loader prepares the chunks for embedding.

4. **Embedding Generation for Documents**  
   - Add an "Embeddings OpenAI1" node.  
   - Select model "text-embedding-3-small".  
   - Connect the output of "Default Data Loader" to this embeddings node.  
   - Set OpenAI API credentials.

5. **Pinecone Vector Store Insertion**  
   - Add a "Pinecone Vector Store" node configured to insert mode.  
   - Select your Pinecone index ("my-docs").  
   - Connect outputs from "Download File From Google Drive" and the embeddings node to this node.  
   - Configure Pinecone API credentials.

6. **Embedding Generation for Queries**  
   - Add an "Embeddings OpenAI" node (same model as above).  
   - Connect this node to "Pinecone Vector Store (Retrieval)" (next step).  
   - Set OpenAI credentials.

7. **Pinecone Vector Store Retrieval**  
   - Add a "Pinecone Vector Store (Retrieval)" node, connected to the embeddings node from step 6.  
   - Use the same Pinecone index and credentials.

8. **Vector Store Tool Setup**  
   - Add a "Vector Store Tool" node.  
   - Configure name "get_documents" and description accordingly.  
   - Connect input from "Pinecone Vector Store (Retrieval)".  

9. **OpenAI Chat Models**  
   - Add two "OpenAI Chat Model" nodes named "OpenAI Chat Model" and "OpenAI Chat Model1".  
   - Set model to "gpt-4o-2024-08-06" with default options.  
   - Assign OpenAI API credentials.

10. **Window Buffer Memory**  
    - Add a "Window Buffer Memory" node.  
    - Set session key to: `docs-{{ $json.messages[0].from }}`  
    - Set sessionIdType to "customKey".  

11. **AI Sales Agent**  
    - Add an "AI Sales Agent" node.  
    - Configure input text as `{{$json.chatInput}}`.  
    - Use system message explaining assistant role and behavior (short factual answers based on internal PDFs).  
    - Connect inputs: chat trigger, memory (Window Buffer Memory), tool (Vector Store Tool), and language model (OpenAI Chat Model).  

12. **Chat Trigger**  
    - Add a "When chat message received" Langchain Chat Trigger node.  
    - Make it public with default options.  
    - Connect output to "AI Sales Agent".

13. **Connect Nodes for Query Flow**  
    - Connect "Embeddings OpenAI" to "Pinecone Vector Store (Retrieval)".  
    - Connect "Pinecone Vector Store (Retrieval)" to "Vector Store Tool".  
    - Connect "Vector Store Tool" to "AI Sales Agent".  
    - Connect "Window Buffer Memory" to "AI Sales Agent".  
    - Connect "OpenAI Chat Model" to "AI Sales Agent".  
    - Connect "OpenAI Chat Model1" to "Vector Store Tool".

14. **Credential Setup**  
    - Configure OpenAI API credentials for all OpenAI nodes.  
    - Configure Pinecone API credentials for Pinecone nodes.  
    - Configure Google Drive OAuth2 credentials for Drive nodes.

15. **Add Sticky Notes**  
    - Add sticky notes near each logical block explaining their purpose and linking to official n8n documentation for Langchain nodes, Pinecone, and Google Drive.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                         | Context or Link                                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Read more about the In-Memory Vector Store                                                                                                          | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreinmemory/          |
| Learn more about using AI Agents                                                                                                                     | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/                         |
| Import your marketing PDF document to build your vector store. This will be used as the knowledgebase by the Tourist guide AI Agent.                | Sticky note near ingestion block                                                                                     |
| n8n's AI agents can remember conversations per individual customer and tap into resources like the product catalogue vector store to pull data.     | Sticky note near AI Sales Agent                                                                                      |

---

**Disclaimer:**  
The provided content is exclusively generated from an n8n automated workflow. All data and documents processed are legal and publicly accessible. This workflow conforms to content policies and contains no illegal, offensive, or protected elements.