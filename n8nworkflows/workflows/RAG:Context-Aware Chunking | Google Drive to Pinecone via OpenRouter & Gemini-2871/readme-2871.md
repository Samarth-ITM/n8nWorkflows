RAG:Context-Aware Chunking | Google Drive to Pinecone via OpenRouter & Gemini

https://n8nworkflows.xyz/workflows/rag-context-aware-chunking---google-drive-to-pinecone-via-openrouter---gemini-2871


# RAG:Context-Aware Chunking | Google Drive to Pinecone via OpenRouter & Gemini

### 1. Workflow Overview

This workflow automates the extraction, contextual chunking, embedding, and storage of document content from Google Drive into a Pinecone vector store. It is designed to enhance Retrieval-Augmented Generation (RAG) by ensuring each text chunk retains meaningful context from the entire document, improving semantic search and AI retrieval accuracy.

**Logical Blocks:**

- **1.1 Input Reception and Document Retrieval:**  
  Triggering the workflow manually and fetching a Google Drive document in plain text format.

- **1.2 Text Extraction and Section Splitting:**  
  Extracting raw text from the document and splitting it into logical sections based on custom boundary markers.

- **1.3 Section Preparation and Looping:**  
  Preparing the split sections for iterative processing by splitting them into individual chunks.

- **1.4 Context Generation via AI Agent:**  
  For each chunk, generating a succinct contextual summary using an AI agent powered by OpenRouter’s GPT-4.0-mini model.

- **1.5 Contextual Chunk Concatenation:**  
  Prepending the generated context to the original chunk text to create enriched content.

- **1.6 Embedding Creation and Storage:**  
  Converting the enriched text chunks into vector embeddings using Google Gemini’s text-embedding-004 model and storing them in Pinecone for efficient retrieval.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Document Retrieval

- **Overview:**  
  This block initiates the workflow manually and downloads a specified Google Drive document as plain text.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Get Document From Google Drive

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow on user command.  
    - *Configuration:* Default manual trigger, no parameters.  
    - *Connections:* Outputs to "Get Document From Google Drive".  
    - *Edge Cases:* None significant; manual trigger ensures controlled execution.

  - **Get Document From Google Drive**  
    - *Type:* Google Drive Node  
    - *Role:* Downloads a Google Docs file as plain text.  
    - *Configuration:*  
      - File ID set to a specific Google Docs document.  
      - Conversion option: Google Docs to "text/plain".  
      - OAuth2 credentials configured for Google Drive access.  
    - *Connections:* Outputs to "Extract Text Data From Google Document".  
    - *Edge Cases:*  
      - Authentication failures if OAuth2 token expires.  
      - File not found or permission errors if file ID is invalid or access is restricted.  
      - Conversion errors if file format changes.

#### 2.2 Text Extraction and Section Splitting

- **Overview:**  
  Extracts raw text from the downloaded document and splits it into logical sections using a custom delimiter.

- **Nodes Involved:**  
  - Extract Text Data From Google Document  
  - Split Document Text Into Sections

- **Node Details:**

  - **Extract Text Data From Google Document**  
    - *Type:* Extract From File  
    - *Role:* Extracts plain text content from the downloaded file.  
    - *Configuration:* Operation set to "text".  
    - *Connections:* Outputs to "Split Document Text Into Sections".  
    - *Edge Cases:*  
      - Extraction failure if file content is corrupted or empty.

  - **Split Document Text Into Sections**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Splits the extracted text into sections based on a custom boundary marker string.  
    - *Configuration:*  
      - Uses a fixed delimiter string:  
        `"—---------------------------—-------------[SECTIONEND]—---------------------------—-------------"`  
      - Splits the text into an array of sections stored in `item.json.section`.  
      - Also stores the JSON stringified sections in `item.json.document`.  
    - *Connections:* Outputs to "Prepare Sections For Looping".  
    - *Edge Cases:*  
      - If the delimiter is missing or malformed, splitting will fail or produce a single section.  
      - Large documents may cause performance issues.

#### 2.3 Section Preparation and Looping

- **Overview:**  
  Prepares the array of sections for iterative processing by splitting them into individual items and looping over each.

- **Nodes Involved:**  
  - Prepare Sections For Looping  
  - Loop Over Items

- **Node Details:**

  - **Prepare Sections For Looping**  
    - *Type:* Split Out  
    - *Role:* Splits the array of sections into individual workflow items for looping.  
    - *Configuration:* Field to split out is `section`.  
    - *Connections:* Outputs to "Loop Over Items".  
    - *Edge Cases:*  
      - Empty or undefined `section` field will cause no iterations.

  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Processes each section chunk individually in a loop.  
    - *Configuration:* Default batch options (processes one item at a time).  
    - *Connections:* Outputs to "AI Agent - Prepare Context".  
    - *Edge Cases:*  
      - Large number of sections may increase runtime.  
      - Batch size misconfiguration could cause performance bottlenecks.

#### 2.4 Context Generation via AI Agent

- **Overview:**  
  Uses an AI agent to generate a succinct contextual summary for each chunk, situating it within the whole document.

- **Nodes Involved:**  
  - AI Agent - Prepare Context

- **Node Details:**

  - **AI Agent - Prepare Context**  
    - *Type:* LangChain Agent Node  
    - *Role:* Generates context metadata for each chunk using GPT-4.0-mini via OpenRouter.  
    - *Configuration:*  
      - Prompt includes the entire document (from `Split Document Text Into Sections` node’s JSON stringified sections) and the current chunk.  
      - Instruction: "Please give a short succinct context to situate this chunk within the overall document for the purposes of improving search retrieval of the chunk. Answer only with the succinct context and nothing else."  
      - Agent type: conversationalAgent.  
      - Credentials: OpenRouter API key configured.  
    - *Connections:* Outputs to "Concatenate the context and section text".  
    - *Edge Cases:*  
      - API rate limits or authentication errors with OpenRouter.  
      - Prompt formatting errors if document or chunk data is missing.  
      - Timeout or incomplete responses from the model.

#### 2.5 Contextual Chunk Concatenation

- **Overview:**  
  Concatenates the AI-generated context with the original chunk text to form a context-rich chunk for embedding.

- **Nodes Involved:**  
  - Concatenate the context and section text

- **Node Details:**

  - **Concatenate the context and section text**  
    - *Type:* Set Node  
    - *Role:* Creates a new field `section_chunk` by combining the AI-generated context and the chunk text.  
    - *Configuration:*  
      - Sets `section_chunk` to:  
        `{{ $json.output }}. {{ $('Loop Over Items').item.json.section }}`  
      - This concatenates the AI context (`$json.output`) with the current chunk text.  
    - *Connections:* Outputs to "Pinecone Vector Store".  
    - *Edge Cases:*  
      - Missing or empty AI context or chunk text will produce incomplete concatenation.

#### 2.6 Embedding Creation and Storage

- **Overview:**  
  Converts the enriched chunks into vector embeddings using Google Gemini and stores them in Pinecone.

- **Nodes Involved:**  
  - Pinecone Vector Store  
  - Embeddings Google Gemini  
  - Default Data Loader  
  - Recursive Character Text Splitter

- **Node Details:**

  - **Pinecone Vector Store**  
    - *Type:* LangChain Vector Store Node  
    - *Role:* Inserts vector embeddings and metadata into Pinecone index.  
    - *Configuration:*  
      - Mode: insert  
      - Pinecone index: "context-rag-test"  
      - Credentials: Pinecone API key configured.  
    - *Connections:*  
      - Receives input from "Concatenate the context and section text".  
      - Outputs back to "Loop Over Items" (to continue looping).  
    - *Edge Cases:*  
      - API authentication failures.  
      - Index not found or misconfigured.  
      - Network timeouts.

  - **Embeddings Google Gemini**  
    - *Type:* LangChain Embeddings Node  
    - *Role:* Converts text into semantic vector embeddings using Google Gemini text-embedding-004.  
    - *Configuration:*  
      - Model: "models/text-embedding-004"  
      - Credentials: Google Palm API key configured.  
    - *Connections:* Outputs to "Pinecone Vector Store" (via ai_embedding channel).  
    - *Edge Cases:*  
      - API quota exceeded or authentication errors.  
      - Model unavailability or version mismatch.

  - **Default Data Loader**  
    - *Type:* LangChain Document Loader  
    - *Role:* Loads documents for embedding processing.  
    - *Configuration:* Default options.  
    - *Connections:* Outputs to "Pinecone Vector Store" (via ai_document channel).  
    - *Edge Cases:* Minimal; depends on input validity.

  - **Recursive Character Text Splitter**  
    - *Type:* LangChain Text Splitter  
    - *Role:* Splits text recursively into chunks up to 100,000 characters (large chunk size).  
    - *Configuration:* Chunk size set to 100,000 characters.  
    - *Connections:* Outputs to "Default Data Loader" (via ai_textSplitter channel).  
    - *Edge Cases:*  
      - Very large text inputs may cause memory or performance issues.

---

### 3. Summary Table

| Node Name                         | Node Type                                  | Functional Role                              | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                  |
|----------------------------------|--------------------------------------------|----------------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger                             | Starts workflow manually                      | -                                | Get Document From Google Drive   |                                                                                                              |
| Get Document From Google Drive    | Google Drive Node                          | Downloads Google Docs file as plain text     | When clicking ‘Test workflow’    | Extract Text Data From Google Document |                                                                                                              |
| Extract Text Data From Google Document | Extract From File Node                    | Extracts plain text from downloaded file     | Get Document From Google Drive   | Split Document Text Into Sections |                                                                                                              |
| Split Document Text Into Sections | Code Node (JavaScript)                     | Splits text into sections by custom delimiter | Extract Text Data From Google Document | Prepare Sections For Looping     |                                                                                                              |
| Prepare Sections For Looping      | Split Out Node                            | Splits array of sections into individual items | Split Document Text Into Sections | Loop Over Items                 |                                                                                                              |
| Loop Over Items                   | Split In Batches                          | Loops over each section chunk                 | Prepare Sections For Looping      | AI Agent - Prepare Context       |                                                                                                              |
| AI Agent - Prepare Context        | LangChain Agent Node                      | Generates succinct context for each chunk    | Loop Over Items                  | Concatenate the context and section text |                                                                                                              |
| Concatenate the context and section text | Set Node                                  | Concatenates AI context with chunk text      | AI Agent - Prepare Context       | Pinecone Vector Store            |                                                                                                              |
| Pinecone Vector Store             | LangChain Vector Store Node               | Stores vector embeddings in Pinecone          | Concatenate the context and section text, Embeddings Google Gemini, Default Data Loader | Loop Over Items                 | ## Convert Text To Vectors In this step, the Pinecone node converts the provided text into vectors using Google Gemini and stores them in the Pinecone vector database. |
| Embeddings Google Gemini          | LangChain Embeddings Node                 | Converts text to vector embeddings            | -                                | Pinecone Vector Store            |                                                                                                              |
| Default Data Loader               | LangChain Document Loader                 | Loads documents for embedding                  | Recursive Character Text Splitter | Pinecone Vector Store            |                                                                                                              |
| Recursive Character Text Splitter | LangChain Text Splitter                   | Splits text into large chunks (up to 100k chars) | -                                | Default Data Loader              |                                                                                                              |
| Sticky Note                      | Sticky Note                               | Describes "Prepare Document" block            | -                                | -                               | ## Prepare Document. This section is responsible for downloading the file from Google Drive, splitting the text into sections by detecting separators, and preparing them for looping. |
| Sticky Note1                     | Sticky Note                               | Describes "Prepare context" block              | -                                | -                               | ## Prepare context In this section, the agent node will prepare context for a section (chunk of text), which will then be passed for conversion into vectors along with the section itself. |
| Sticky Note2                     | Sticky Note                               | Describes "Convert Text To Vectors" block     | -                                | -                               | ## Convert Text To Vectors In this step, the Pinecone node converts the provided text into vectors using Google Gemini and stores them in the Pinecone vector database. |
| Sticky Note3                     | Sticky Note                               | Video demo link and thumbnail                   | -                                | -                               | ## Video Demo [![Video Thumbnail](https://img.youtube.com/vi/qBeWP65I4hg/maxresdefault.jpg)](https://www.youtube.com/watch?v=qBeWP65I4hg) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a "Manual Trigger" node named "When clicking ‘Test workflow’".  
   - No special configuration needed.

2. **Add Google Drive Node:**  
   - Add "Google Drive" node named "Get Document From Google Drive".  
   - Set operation to "download".  
   - Set file ID to the target Google Docs document ID.  
   - Under options, enable file conversion: convert Google Docs to "text/plain".  
   - Configure Google Drive OAuth2 credentials with appropriate scopes.

3. **Add Extract From File Node:**  
   - Add "Extract From File" node named "Extract Text Data From Google Document".  
   - Set operation to "text".  
   - Connect output of Google Drive node to this node.

4. **Add Code Node for Section Splitting:**  
   - Add "Code" node named "Split Document Text Into Sections".  
   - Use JavaScript code to split the extracted text by the delimiter string:  
     ```
     let split_text = "—---------------------------—-------------[SECTIONEND]—---------------------------—-------------";
     for (const item of $input.all()) {
       item.json.section = item.json.data.split(split_text);
       item.json.document = JSON.stringify(item.json.section);
     }
     return $input.all();
     ```  
   - Connect output of Extract From File node to this node.

5. **Add Split Out Node:**  
   - Add "Split Out" node named "Prepare Sections For Looping".  
   - Set field to split out as `section`.  
   - Connect output of Code node to this node.

6. **Add Split In Batches Node:**  
   - Add "Split In Batches" node named "Loop Over Items".  
   - Use default batch size (1).  
   - Connect output of Split Out node to this node.

7. **Add LangChain Agent Node:**  
   - Add "Agent" node named "AI Agent - Prepare Context".  
   - Set prompt text to:  
     ```
     =<document> 
     {{ $('Split Document Text Into Sections').item.json.document }}
     </document> 
     Here is the chunk we want to situate within the whole document 
     <chunk> 
     {{ $json.section }}
     </chunk> 
     Please give a short succinct context to situate this chunk within the overall document for the purposes of improving search retrieval of the chunk. Answer only with the succinct context and nothing else.
     ```  
   - Select agent type "conversationalAgent".  
   - Configure OpenRouter API credentials.  
   - Connect output of "Loop Over Items" node to this node.

8. **Add Set Node for Concatenation:**  
   - Add "Set" node named "Concatenate the context and section text".  
   - Create a new string field `section_chunk` with value:  
     `={{ $json.output }}. {{ $('Loop Over Items').item.json.section }}`  
   - Connect output of Agent node to this node.

9. **Add LangChain Embeddings Node:**  
   - Add "Embeddings Google Gemini" node.  
   - Set model name to "models/text-embedding-004".  
   - Configure Google Palm API credentials.  
   - Connect output of "Concatenate the context and section text" node to this node.

10. **Add LangChain Document Loader Node:**  
    - Add "Default Data Loader" node with default options.  
    - Connect output of "Recursive Character Text Splitter" node to this node.

11. **Add LangChain Text Splitter Node:**  
    - Add "Recursive Character Text Splitter" node.  
    - Set chunk size to 100,000 characters.  
    - Connect output of "Embeddings Google Gemini" node to this node.

12. **Add Pinecone Vector Store Node:**  
    - Add "Pinecone Vector Store" node.  
    - Set mode to "insert".  
    - Select or create Pinecone index named "context-rag-test".  
    - Configure Pinecone API credentials.  
    - Connect output of "Concatenate the context and section text" node to this node (main input).  
    - Connect output of "Embeddings Google Gemini" node to this node (ai_embedding input).  
    - Connect output of "Default Data Loader" node to this node (ai_document input).  
    - Connect output of "Recursive Character Text Splitter" node to this node (ai_textSplitter input).  
    - Connect output of Pinecone node back to "Loop Over Items" node to continue looping.

13. **Connect Workflow:**  
    - Ensure all nodes are connected as per the above steps to maintain data flow.

14. **Add Sticky Notes (Optional):**  
    - Add sticky notes to document workflow sections for clarity, using the content from the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow based on the article: https://www.anthropic.com/news/contextual-retrieval                                      | Source article describing context-aware retrieval techniques                                    |
| Video demo available: [YouTube Video](https://www.youtube.com/watch?v=qBeWP65I4hg)                                      | Video walkthrough of the workflow with visual explanation                                       |
| This workflow enhances RAG retrieval accuracy by ensuring chunks retain meaningful context, improving AI response quality | Use case: semantic search, AI knowledge management, intelligent document retrieval              |

---

This document provides a detailed, structured reference to understand, reproduce, and modify the "RAG:Context-Aware Chunking | Google Drive to Pinecone via OpenRouter & Gemini" workflow, including all nodes, configurations, and potential edge cases.