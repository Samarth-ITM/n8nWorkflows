Optimize Unstructured Data for RAG with Blockify IdeaBlocks Technology

https://n8nworkflows.xyz/workflows/optimize-unstructured-data-for-rag-with-blockify-ideablocks-technology-7475


# Optimize Unstructured Data for RAG with Blockify IdeaBlocks Technology

### 1. Workflow Overview

This workflow, titled **"Optimize Unstructured Data for RAG with Blockify IdeaBlocks Technology,"** is designed to process large unstructured text data, optimize it into structured "IdeaBlocks" using Blockify’s API, embed these IdeaBlocks into a vector store, and enable retrieval-augmented generation (RAG) through an AI chatbot interface. The core purpose is to improve the accuracy and efficiency of AI language models when querying large knowledge sets by structuring and chunking the input data intelligently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Text Extraction**: Retrieves raw text data from uploaded files (Google Drive .TXT file) and extracts the text content.
- **1.2 Text Chunking**: Splits extracted raw text into overlapping chunks of manageable size for processing.
- **1.3 Blockify Ingest API Processing**: Sends each chunk to the Blockify API to convert raw text into structured “IdeaBlocks” XML.
- **1.4 IdeaBlocks Storage and Embedding**: Processes the API response, extracts IdeaBlocks, and stores them in an in-memory vector store with embeddings.
- **1.5 AI Agent and RAG Chatbot Interaction**: Configures an AI agent that uses the embedded IdeaBlocks to answer user queries via a chatbot interface, enabling RAG capabilities.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Text Extraction

- **Overview:**  
  This block downloads a .TXT file from Google Drive and extracts the raw text content for further processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Download .TXT File for Ingest (Google Drive node)  
  - Extract Text from .TXT File (ExtractFromFile node)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Connections: Outputs to "Download .TXT File for Ingest" node.  
    - Edge cases: No authentication needed, but execution depends on manual trigger.

  - **Download .TXT File for Ingest**  
    - Type: Google Drive (download operation)  
    - Role: Downloads the specified .TXT file by fileId from Google Drive.  
    - Configuration: Uses OAuth2 credentials for Google Drive access; fileId set to a specific file containing source text.  
    - Inputs: Triggered by manual trigger node.  
    - Outputs: Raw file data to "Extract Text from .TXT File."  
    - Edge cases: File not found, permission denied, token expiration.

  - **Extract Text from .TXT File**  
    - Type: ExtractFromFile  
    - Role: Extracts text content from the downloaded file.  
    - Configuration: Operation set to "text" extraction.  
    - Inputs: Receives file data from Google Drive node.  
    - Outputs: Extracted text to "Chunk Text" node.  
    - Edge cases: Unsupported file type, corrupted file content.

---

#### 2.2 Text Chunking

- **Overview:**  
  This block splits large raw text into smaller chunks with overlap to prepare for processing by the Blockify API, optimizing context retention.

- **Nodes Involved:**  
  - Chunk Text (Code node)  
  - Loop Over Chunks (SplitInBatches node)  

- **Node Details:**

  - **Chunk Text**  
    - Type: Code (JavaScript)  
    - Role: Splits input text into chunks of max 1000 characters with 100 characters overlap, respecting sentence boundaries.  
    - Configuration: Custom JS function that finds natural breakpoints (punctuation or newline) before max chunk length.  
    - Inputs: Text from "Extract Text from .TXT File."  
    - Outputs: Multiple chunked text items, each with chunk index and chunk text.  
    - Edge cases: Null or empty text input, very short texts, irregular punctuation.

  - **Loop Over Chunks**  
    - Type: SplitInBatches  
    - Role: Processes chunks in batches, enabling iterative processing for downstream API calls.  
    - Configuration: Default options, processes chunks one by one or in batches.  
    - Inputs: Chunked texts from "Chunk Text."  
    - Outputs: Batches to "Blockify Ingest API."  
    - Edge cases: Batch size misconfiguration, empty input batches.

---

#### 2.3 Blockify Ingest API Processing

- **Overview:**  
  Sends each chunk of text to the Blockify Ingest API, which returns structured IdeaBlocks XML, transforming unstructured text into semantically meaningful blocks.

- **Nodes Involved:**  
  - Blockify Ingest API (HTTP Request node)  
  - Extract IdeaBlocks from API Response (Set node)  

- **Node Details:**

  - **Blockify Ingest API**  
    - Type: HTTP Request  
    - Role: Calls the Blockify API endpoint with the chunk text as input.  
    - Configuration:  
      - POST method to https://api.blockify.ai/v1/chat/completions  
      - JSON body includes the chunk text as user content message  
      - Model set to "ingest" with max_tokens 8000 and temperature 0.5  
      - Authentication: HTTP Bearer Token (Blockify API key)  
    - Inputs: Receives batches of text chunks from "Loop Over Chunks."  
    - Outputs: API JSON response with IdeaBlocks content to "Extract IdeaBlocks from API Response."  
    - Edge cases: API key invalid, rate limiting, network timeouts, API errors.

  - **Extract IdeaBlocks from API Response**  
    - Type: Set  
    - Role: Extracts the raw IdeaBlocks XML content from the API response JSON path `choices[0].message.content`.  
    - Configuration: Uses expression to assign the content string.  
    - Inputs: API response JSON from "Blockify Ingest API."  
    - Outputs: Clean IdeaBlocks XML content to "Simple IdeaBlock Vector Store."  
    - Edge cases: Unexpected API response structure, missing fields.

---

#### 2.4 IdeaBlocks Storage and Embedding

- **Overview:**  
  This block stores the processed IdeaBlocks into an in-memory vector store and creates embeddings for later retrieval by the AI agent.

- **Nodes Involved:**  
  - Simple IdeaBlock Vector Store (@n8n/n8n-nodes-langchain.vectorStoreInMemory)  
  - Embeddings OpenAI (@n8n/n8n-nodes-langchain.embeddingsOpenAi)  
  - Default Data Loader (@n8n/n8n-nodes-langchain.documentDefaultDataLoader)  
  - Embed Individual IdeaBlocks (Already Separated) (@n8n/n8n-nodes-langchain.textSplitterCharacterTextSplitter)  

- **Node Details:**

  - **Simple IdeaBlock Vector Store**  
    - Type: Vector Store In-Memory (insert mode)  
    - Role: Inserts new data items (IdeaBlocks) into an in-memory vector store with a shared memory key.  
    - Configuration: Uses a memory key list `vector_store_key` to track stored vectors.  
    - Inputs: Receives IdeaBlocks XML chunks from "Extract IdeaBlocks from API Response" and loops back from "Loop Over Chunks" (for batch processing). Also connected to data loaders and embeddings.  
    - Outputs: Loops chunks back to "Loop Over Chunks" for continuous ingestion cycle.  
    - Edge cases: Memory limits, data duplication, concurrency conflicts.

  - **Embeddings OpenAI**  
    - Type: OpenAI Embeddings node  
    - Role: Generates vector embeddings for text data (IdeaBlocks) using OpenAI's embedding models.  
    - Configuration: Default embedding options; uses OpenAI API credentials.  
    - Inputs: Receives text from vector store or directly for embedding.  
    - Outputs: Embedding vectors passed to "Simple IdeaBlock Vector Store" and "Query Data Tool."  
    - Edge cases: API quota exceeded, network errors, invalid API key.

  - **Default Data Loader**  
    - Type: Document Default Data Loader  
    - Role: Loads and processes documents with custom text splitting mode.  
    - Inputs: Receives text split by "Embed Individual IdeaBlocks."  
    - Outputs: Passes loaded documents to "Simple IdeaBlock Vector Store."  
    - Edge cases: Data format errors, empty input.

  - **Embed Individual IdeaBlocks (Already Separated)**  
    - Type: Text Splitter (Character Text Splitter)  
    - Role: Splits IdeaBlocks XML string by separator `</ideablock>` to isolate individual IdeaBlock elements.  
    - Configuration: Chunk size set very large (1,000,000) to allow splitting only by separator.  
    - Inputs: Receives raw IdeaBlocks XML from "Default Data Loader."  
    - Outputs: Individual IdeaBlocks to "Default Data Loader."  
    - Edge cases: Malformed XML, missing separator tags.

---

#### 2.5 AI Agent and RAG Chatbot Interaction

- **Overview:**  
  Implements an AI chatbot interface that retrieves relevant IdeaBlocks from the vector store using a Langchain AI agent, enabling retrieval-augmented generation (RAG) powered conversation.

- **Nodes Involved:**  
  - RAG Chatbot (@n8n/n8n-nodes-langchain.chatTrigger)  
  - AI Agent (@n8n/n8n-nodes-langchain.agent)  
  - Query Data Tool (@n8n/n8n-nodes-langchain.vectorStoreInMemory)  
  - OpenAI Chat Model (@n8n/n8n-nodes-langchain.lmChatOpenAi)  

- **Node Details:**

  - **RAG Chatbot**  
    - Type: Chat Trigger (Webhook-based chat interface)  
    - Role: Public-facing chatbot UI for users to submit queries about n8n and receive AI-generated answers.  
    - Configuration:  
      - Public enabled webhook  
      - Custom CSS for glassmorphism theme and styling  
      - Placeholder text and initial greeting message provided  
    - Inputs: Receives user chat messages via webhook.  
    - Outputs: Sends chat messages to "AI Agent."  
    - Edge cases: Webhook connectivity, input validation, UI rendering issues.

  - **AI Agent**  
    - Type: Langchain Agent  
    - Role: Orchestrates query processing by combining language model and vector store retrieval tools.  
    - Configuration: Uses OpenAI chat model and vector store as tools.  
    - Inputs: Receives chat messages from "RAG Chatbot."  
    - Outputs: Responds back to "RAG Chatbot."  
    - Edge cases: Language model errors, tool retrieval failures.

  - **Query Data Tool**  
    - Type: Vector Store In-Memory (retrieve-as-tool mode)  
    - Role: Retrieves top 5 relevant IdeaBlocks from the vector store to support AI agent responses.  
    - Configuration: Tool name "knowledge_base", memory key `vector_store_key`, topK=5 results.  
    - Inputs: Invoked as a tool by "AI Agent."  
    - Outputs: Returns relevant vector data for language model.  
    - Edge cases: Empty vector store, retrieval inaccuracies.

  - **OpenAI Chat Model**  
    - Type: OpenAI Chat  
    - Role: Language model to generate chat responses based on retrieved context and user input.  
    - Configuration: Model set to "gpt-4.1-nano" (cached), default options.  
    - Inputs: Used by "AI Agent" as language model.  
    - Outputs: Text generation for chatbot.  
    - Edge cases: Model availability, token limits, API errors.

---

### 3. Summary Table

| Node Name                             | Node Type                                      | Functional Role                             | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                                                              |
|-------------------------------------|------------------------------------------------|--------------------------------------------|----------------------------------|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’    | Manual Trigger                                 | Entry point to start workflow manually     | -                                | Download .TXT File for Ingest          |                                                                                                                                          |
| Download .TXT File for Ingest       | Google Drive (download)                         | Downloads source .TXT file from Google Drive | When clicking ‘Execute workflow’  | Extract Text from .TXT File             | ## Collect Source Documents\n\nExtract data and convert into text through any traditional methods so the information can be passed into the Blockify text based API.\n\nThis sample uses a simple .TXT file uploaded to Google Drive, but the content could be anything. |
| Extract Text from .TXT File          | ExtractFromFile                                | Extracts plain text from downloaded file   | Download .TXT File for Ingest     | Chunk Text                            |                                                                                                                                          |
| Chunk Text                          | Code                                           | Splits text into chunks with overlap       | Extract Text from .TXT File       | Loop Over Chunks                      | ## Chunk Text\n\nChunk using any normal chunking method. For optimal results we recommend sending between 1,000 and 4,000 Character chunks per Blockify Ingest API request with a 10% Overlap.\n\nThis example uses 1,000 Characters and 100 Character Overlap. |
| Loop Over Chunks                    | SplitInBatches                                 | Processes chunks in batches for API calls  | Chunk Text                      | Blockify Ingest API (batch input), Loop Over Chunks (batch output) |                                                                                                                                          |
| Blockify Ingest API                 | HTTP Request                                   | Sends chunk text to Blockify API for structuring | Loop Over Chunks                | Extract IdeaBlocks from API Response   | ## Blockify Ingest API\n\nGet your free trial API key here: https://console.blockify.ai/signup\n\nBlockify Ingest is designed to receive input raw source (parsed / chunked) text via LLM API request and output structured optimized XML “IdeaBlocks,” repackaging the source data into a cleaned format. \n\nThe process is ≈99% lossless for numerical data, facts, and key information. We always encourage a human to be in the loop to review the outputs of IdeaBlock content. |
| Extract IdeaBlocks from API Response | Set                                            | Extracts IdeaBlocks XML content from API response | Blockify Ingest API             | Simple IdeaBlock Vector Store          |                                                                                                                                          |
| Simple IdeaBlock Vector Store      | Vector Store In-Memory (insert mode)           | Stores IdeaBlocks data as vectors in memory | Extract IdeaBlocks from API Response, Loop Over Chunks, Default Data Loader | Loop Over Chunks, Query Data Tool       | ## Embed IdeaBlocks\n\nRecommend testing different elements of the XML to Embed \n\nHow you split and load the IdeaBlocks into your Vector DB is at your discretion. For optimal result try different combinations of the XML data for each IdeaBlock.\n\nI.e. Just embedding the Critical Question, vs Critical Question + Trusted Answer, etc. \n\nRemember to serve at a minimum the Critical Question and the Trusted Answer in the TEXT part of the DB entry. Some use cases benefit from the embedding text being differed from the text returned during the search. |
| Embed Individual IdeaBlocks (Already Separated) | Text Splitter (CharacterTextSplitter)         | Splits IdeaBlocks XML by closing tag        | Default Data Loader              | Default Data Loader                   |                                                                                                                                          |
| Default Data Loader                | Document Default Data Loader                    | Loads and processes IdeaBlock documents     | Embed Individual IdeaBlocks     | Simple IdeaBlock Vector Store          |                                                                                                                                          |
| Embeddings OpenAI                 | OpenAI Embeddings                               | Generates embeddings for IdeaBlocks         | Simple IdeaBlock Vector Store   | Simple IdeaBlock Vector Store, Query Data Tool |                                                                                                                                          |
| Query Data Tool                   | Vector Store In-Memory (retrieve-as-tool mode) | Retrieves top-K relevant IdeaBlocks for AI Agent | AI Agent                      | AI Agent                              | ## Basic RAG\n\nUsed to simply showcase how the IdeaBlocks can be used. IdeaBlocks could be used in place of any Chunk data source approach (RAG or otherwise). |
| AI Agent                        | Langchain Agent                                  | Coordinates retrieval and language model to answer queries | RAG Chatbot, Query Data Tool, OpenAI Chat Model | RAG Chatbot                        |                                                                                                                                          |
| OpenAI Chat Model                | OpenAI Chat Model                               | Provides chat completions with GPT-4.1-nano | AI Agent                      | AI Agent                              |                                                                                                                                          |
| RAG Chatbot                     | Chat Trigger (Chatbot UI)                        | Public chatbot interface for user queries   | User via webhook               | AI Agent                             |                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: To manually start the workflow.

2. **Add a Google Drive Node for Downloading Source File**  
   - Name: "Download .TXT File for Ingest"  
   - Operation: Download  
   - File ID: Set to your target .TXT file’s Google Drive ID  
   - Credentials: Configure Google Drive OAuth2 credentials.

3. **Add ExtractFromFile Node**  
   - Name: "Extract Text from .TXT File"  
   - Operation: Text extraction  
   - Connect input from "Download .TXT File for Ingest."

4. **Add a Code Node to Chunk Text**  
   - Name: "Chunk Text"  
   - Paste the provided JavaScript code (splitting text into ~1000 character chunks with 100 character overlap).  
   - Connect input from "Extract Text from .TXT File."

5. **Add a SplitInBatches Node**  
   - Name: "Loop Over Chunks"  
   - Default batch size (1 or more) to process chunks one at a time.  
   - Connect input from "Chunk Text."

6. **Add HTTP Request Node for Blockify Ingest API**  
   - Name: "Blockify Ingest API"  
   - Method: POST  
   - URL: `https://api.blockify.ai/v1/chat/completions`  
   - Authentication: HTTP Bearer Token (create credential with your Blockify API key)  
   - Body (JSON):  
     ```json
     {
       "messages": [{"role": "user", "content": "{{ JSON.stringify($json.chunk) }}"}],
       "max_tokens": 8000,
       "temperature": 0.5,
       "model": "ingest"
     }
     ```  
   - Connect input from "Loop Over Chunks" (batch output).

7. **Add a Set Node to Extract API Response**  
   - Name: "Extract IdeaBlocks from API Response"  
   - Set field `choices[0].message.content` using expression `{{$json["choices"][0]["message"]["content"]}}`  
   - Connect input from "Blockify Ingest API."

8. **Add Vector Store In-Memory Node (Insert Mode)**  
   - Name: "Simple IdeaBlock Vector Store"  
   - Mode: Insert  
   - Memory Key: Use a consistent key, e.g., `vector_store_key` (list mode)  
   - Connect input from "Extract IdeaBlocks from API Response."  
   - Also connect loop output back to "Loop Over Chunks" to enable batch processing continuation.

9. **Add Text Splitter Node (Character Text Splitter)**  
   - Name: "Embed Individual IdeaBlocks (Already Separated)"  
   - Chunk Size: Very large (e.g., 1,000,000 characters)  
   - Separator: `</ideablock>`  
   - Connect input from a data loader (see next step).

10. **Add Document Default Data Loader Node**  
    - Name: "Default Data Loader"  
    - Text Splitting Mode: Custom  
    - Connect input from "Embed Individual IdeaBlocks (Already Separated)" (output).

11. **Connect "Default Data Loader" output to "Simple IdeaBlock Vector Store"**  
    - This completes the ingestion and embedding pipeline.

12. **Add OpenAI Embeddings Node**  
    - Name: "Embeddings OpenAI"  
    - Use OpenAI API credentials  
    - Connect input from "Simple IdeaBlock Vector Store" for embedding generation.  
    - Connect output back to "Simple IdeaBlock Vector Store" and to "Query Data Tool."

13. **Add Vector Store In-Memory Node (Retrieve-as-Tool Mode)**  
    - Name: "Query Data Tool"  
    - Mode: Retrieve as Tool  
    - Tool Name: "knowledge_base"  
    - TopK: 5  
    - Memory Key: Same `vector_store_key` as used previously  
    - Connect input from AI Agent (see next step).

14. **Add OpenAI Chat Model Node**  
    - Name: "OpenAI Chat Model"  
    - Model: "gpt-4.1-nano" or equivalent  
    - Credentials: OpenAI API key  
    - Connect input from AI Agent node.

15. **Add Langchain Agent Node**  
    - Name: "AI Agent"  
    - Configure with OpenAI Chat Model and Query Data Tool as tools for retrieval and language generation.  
    - Connect input from "RAG Chatbot."  
    - Connect outputs back to "RAG Chatbot."

16. **Add Chat Trigger Node**  
    - Name: "RAG Chatbot"  
    - Configure as public webhook with custom CSS and placeholder text for UI.  
    - Connect input from user via webhook.  
    - Connect output to "AI Agent."

17. **Connect "When clicking ‘Execute workflow’" to "Download .TXT File for Ingest"**  
    - This completes the manual trigger chain.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Blockify® Data Optimization Workflow: Improves accuracy for RAG by structuring unstructured data into small "IdeaBlocks" that increase LLM accuracy by ~78X and reduce data size to ~2.5%. It is particularly effective in domains like healthcare and sales meeting transcripts. Learn more at [https://iternal.ai/blockify](https://iternal.ai/blockify) and sign up for free demo API access at [https://console.blockify.ai/signup](https://console.blockify.ai/signup). The technical whitepaper and accuracy studies are also available online. | Branding and project overview: https://iternal.ai/blockify                                        |
| Collect source documents by converting any file type (Word, PDF, slides, images) into text for ingestion. This sample uses a .TXT file from Google Drive as an example.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note on Input Reception block                                                                |
| Chunk text with overlap to maintain context. For best results, 1,000 to 4,000 characters per chunk with ~10% overlap is recommended. This example uses 1,000 characters and 100 overlap.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note on Text Chunking block                                                                 |
| Blockify Ingest API processes raw chunks into structured XML IdeaBlocks, preserving data with ≈99% accuracy. Human review is recommended to ensure quality.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note on Blockify Ingest API block                                                           |
| Embedding IdeaBlocks: Test different XML elements for embedding (e.g., Critical Question alone vs. combined with Trusted Answer). Always embed at least the Critical Question and Trusted Answer for best search relevance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note on IdeaBlocks Embedding block                                                          |
| Basic RAG example: Demonstrates how IdeaBlocks can replace standard chunk data sources in retrieval-augmented generation workflows.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note on AI Agent / Query Data Tool block                                                    |

---

**Disclaimer:** The provided document is generated exclusively from an automated n8n workflow export and follows all applicable content policies. The workflow handles only legal and public data sources.