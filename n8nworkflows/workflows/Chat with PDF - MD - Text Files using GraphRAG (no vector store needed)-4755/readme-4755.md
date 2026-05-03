Chat with PDF / MD / Text Files using GraphRAG (no vector store needed)

https://n8nworkflows.xyz/workflows/chat-with-pdf---md---text-files-using-graphrag--no-vector-store-needed--4755


# Chat with PDF / MD / Text Files using GraphRAG (no vector store needed)

---

### 1. Workflow Overview

This workflow enables users to **upload PDF, Markdown (MD), or plain text files from a specified Google Drive folder into an InfraNodus knowledge graph** (GraphRAG), then **chat interactively with the content of these files** using InfraNodus as a knowledge base instead of a traditional vector store. The workflow is divided into two main logical blocks:

- **1.1 Document Ingestion and Processing**:  
  Scans a target Google Drive folder, downloads files, detects their type (PDF, Markdown, text, etc.), extracts text content accordingly, and uploads the textual data to InfraNodus as a knowledge graph.

- **1.2 Interactive Chat Using InfraNodus GraphRAG**:  
  Listens for chat messages (disabled in this version, but designed for interactivity), invokes an AI agent that queries the InfraNodus graph knowledge base, and returns responses enriched by the graph-based retrieval-augmented generation (GraphRAG).

This structure avoids traditional vector stores by leveraging InfraNodus’ graph-based representation and API, simplifying knowledge base setup and enhancing topical understanding.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion and Processing

- **Overview:**  
  This block ingests documents from a specified Google Drive folder, identifies their file types, extracts their textual content appropriately, and sends the text data to InfraNodus to build or update a knowledge graph. This allows the knowledge base to be constructed iteratively from user-uploaded files.

- **Nodes Involved:**  
  - Click ‘Test workflow’ to ingest the documents (Manual Trigger)  
  - Search Google Drive (Google Drive)  
  - Loop Over Items (SplitInBatches)  
  - Retrieve File (Google Drive)  
  - Switch (Switch node for MIME type branching)  
  - Extract from PDF (ExtractFromFile)  
  - Extract from Text File (ExtractFromFile)  
  - Extract from Markdown (ExtractFromFile)  
  - Map PDF to Text (Set)  
  - InfraNodus Save to Graph (HTTP Request)  
  - Sticky Note nodes: Sticky Note4, Sticky Note5, Sticky Note1 (documentation and instructions shown in editor)

- **Node Details:**

  1. **Click ‘Test workflow’ to ingest the documents**  
     - Type: Manual Trigger  
     - Role: Entry point to start the ingestion manually.  
     - Connections: Outputs to "Search Google Drive".  
     - Edge Cases: None; manual trigger must be invoked explicitly.

  2. **Search Google Drive**  
     - Type: Google Drive (fileFolder resource)  
     - Role: Lists all files in a specified Google Drive folder (ID: `13tqp0SaI_v4jG1CFLAZo96isx-UBno4v` named "GraphRAG").  
     - Configuration: Returns all files in folder; query string is "*".  
     - Credentials: Uses Google Drive OAuth2.  
     - Connections: Outputs to "Loop Over Items".  
     - Edge Cases: Empty folder returns no files; auth failure if token expired.

  3. **Loop Over Items**  
     - Type: SplitInBatches  
     - Role: Processes files one by one or in small batches to avoid overloading downstream nodes.  
     - Connections: Two outputs — first output (empty) and second output connects to "Retrieve File".  
     - Edge Cases: Large file sets may slow processing; batch size defaults.

  4. **Retrieve File**  
     - Type: Google Drive (download operation)  
     - Role: Downloads the actual file binary content using file ID from previous step.  
     - Connections: Output goes to "Switch".  
     - Edge Cases: File not found, permission denied, network issues.

  5. **Switch**  
     - Type: Switch Node (MIME type based branching)  
     - Role: Routes files based on MIME type to appropriate extraction nodes:  
       - PDF: `application/pdf`  
       - Text: `text/plain`  
       - Markdown: `text/markdown`  
       - JSON, docs, csv: configured but no outputs connected (ignored in this workflow)  
     - Connections:  
       - PDF → Extract from PDF  
       - Text → Extract from Text File  
       - Markdown → Extract from Markdown  
     - Edge Cases: Unsupported MIME types fall through without processing.

  6. **Extract from PDF**  
     - Type: ExtractFromFile  
     - Role: Extracts textual content from PDF binaries.  
     - Connection: Output to "Map PDF to Text".  
     - Edge Cases: Complex PDFs with images or scanned pages may yield poor text extraction.

  7. **Map PDF to Text**  
     - Type: Set  
     - Role: Maps extracted text to property `data` for downstream HTTP request.  
     - Connection: Output to "InfraNodus Save to Graph".  
     - Edge Cases: Empty or null text could cause downstream issues.

  8. **Extract from Text File**  
     - Type: ExtractFromFile  
     - Role: Extracts text from plain text files.  
     - Connection: Output to "InfraNodus Save to Graph".  

  9. **Extract from Markdown**  
     - Type: ExtractFromFile  
     - Role: Extracts text from Markdown files (treated as text).  
     - Connection: Output to "InfraNodus Save to Graph".  

  10. **InfraNodus Save to Graph**  
      - Type: HTTP Request  
      - Role: Sends extracted text to InfraNodus API to save/update a knowledge graph named "graphrag_from_google_drive".  
      - Method: POST to InfraNodus API endpoint `/graphAndStatements` with parameters including text, graph name, categories (file name), and context settings.  
      - Authentication: HTTP Bearer Auth with InfraNodus DeeMeeTree API key.  
      - Connection: Outputs back to "Loop Over Items" to continue batching.  
      - Edge Cases: API failures, network timeouts, invalid API key, rate limits.  
      - Notes: The graph name must be consistent with chat query graph for knowledge coherence.

  11. **Sticky Note4, Sticky Note5, Sticky Note1**  
      - Type: Sticky Note  
      - Role: Provides detailed user instructions embedded in the workflow editor, including InfraNodus usage, API key setup, and optional PDF conversion tips.  
      - Content includes links to InfraNodus and ConvertAPI services for enhancing workflow usage.

  12. **Convert File to PDF** (disabled)  
      - Type: HTTP Request (ConvertAPI)  
      - Role: Optional better PDF-to-text conversion node using ConvertAPI service instead of default extraction. Disabled by default.  
      - Edge Cases: Requires ConvertAPI account and API key; disabled in this workflow version.

---

#### 2.2 Interactive Chat Using InfraNodus GraphRAG

- **Overview:**  
  This block listens for chat messages (currently disabled) and uses Langchain AI nodes to query the InfraNodus knowledge graph, applying a GraphRAG approach. The AI agent combines conversational memory and OpenAI GPT-4o-mini model to generate informed responses leveraging the knowledge base.

- **Nodes Involved:**  
  - When chat message received (Langchain Chat Trigger) [disabled]  
  - AI Agent (Langchain Agent)  
  - Simple Memory (Langchain Memory Buffer Window)  
  - OpenAI Chat Model (Langchain LM Chat OpenAi)  
  - Knowledge Base GraphRAG (HTTP RequestTool to InfraNodus API)  
  - Sticky Note (instructions for chat usage)

- **Node Details:**

  1. **When chat message received**  
     - Type: Langchain Chat Trigger  
     - Role: Webhook trigger for receiving chat messages from users.  
     - Disabled: True (needs activation for live use).  
     - Parameters: Public access enabled; initial prompt encourages questions about PDFs or commands like `/question`.  
     - Output: To "AI Agent".  
     - Edge Cases: Requires proper webhook setup and public endpoint exposure.

  2. **AI Agent**  
     - Type: Langchain Agent  
     - Role: Central AI node orchestrating response generation. Uses system message to instruct it to use attached knowledge base.  
     - Inputs: Receives chat trigger input, memory, language model, and tool outputs.  
     - Outputs: Delivers final AI responses.  
     - Parameters: System message emphasizes using the knowledge base.  
     - Edge Cases: Misconfiguration may cause irrelevant answers; depends on correct API keys and connected nodes.

  3. **Simple Memory**  
     - Type: Langchain Memory Buffer Window  
     - Role: Maintains conversational context window for chat continuity.  
     - Connected as AI memory input to AI Agent.  
     - Edge Cases: Limited memory size may lose long-term context.

  4. **OpenAI Chat Model**  
     - Type: Langchain LM Chat OpenAI  
     - Role: GPT-4o-mini model used for language generation.  
     - Credentials: OpenAI API key required.  
     - Outputs to AI Agent as language model.  
     - Edge Cases: API quota limits, model availability, latency.

  5. **Knowledge Base GraphRAG**  
     - Type: HTTP Request Tool  
     - Role: Queries InfraNodus knowledge graph with user prompt to retrieve relevant information and graph summary.  
     - Method: POST to InfraNodus API `/graphAndAdvice` with parameters including graph name, prompt, and request mode.  
     - Authentication: HTTP Bearer Auth with InfraNodus API key.  
     - Connected as AI tool input to AI Agent.  
     - Edge Cases: API failures, rate limits, incomplete or empty graph.

  6. **Sticky Note** (Step 2 instructions)  
     - Provides guidance on activating the chat nodes, configuring InfraNodus API key, naming the graph consistently, and tips for multi-knowledge base setups.

---

### 3. Summary Table

| Node Name                        | Node Type                      | Functional Role                            | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                    |
|---------------------------------|--------------------------------|--------------------------------------------|----------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------|
| Click ‘Test workflow’ to ingest the documents | Manual Trigger                | Entry point to start Google Drive ingestion |                            | Search Google Drive       |                                                                                                               |
| Search Google Drive              | Google Drive (fileFolder)       | Lists all files in target Google Drive folder | Click ‘Test workflow’      | Loop Over Items           |                                                                                                               |
| Loop Over Items                 | SplitInBatches                 | Processes files in batches                   | Search Google Drive         | Retrieve File, Loop Over Items (loop continue) |                                                                                                               |
| Retrieve File                   | Google Drive (download)          | Downloads file content                       | Loop Over Items             | Switch                   |                                                                                                               |
| Switch                         | Switch                        | Routes files by MIME type                    | Retrieve File               | Extract from PDF, Extract from Text File, Extract from Markdown |                                                                                                               |
| Extract from PDF               | ExtractFromFile                 | Extracts text from PDF files                  | Switch (pdf output)         | Map PDF to Text           |                                                                                                               |
| Map PDF to Text                | Set                           | Maps extracted PDF text to property "data"  | Extract from PDF            | InfraNodus Save to Graph  |                                                                                                               |
| Extract from Text File         | ExtractFromFile                 | Extracts text from plain text files           | Switch (text output)        | InfraNodus Save to Graph  |                                                                                                               |
| Extract from Markdown          | ExtractFromFile                 | Extracts text from Markdown files             | Switch (md output)          | InfraNodus Save to Graph  |                                                                                                               |
| InfraNodus Save to Graph       | HTTP Request                   | Sends text data to InfraNodus to update graph | Map PDF to Text, Extract from Text File, Extract from Markdown | Loop Over Items           |                                                                                                               |
| Convert File to PDF             | HTTP Request (disabled)         | Optional better PDF to text conversion via ConvertAPI |                            |                          | Optional: Better PDF Conversion using ConvertAPI (disabled by default)                                        |
| Sticky Note4                   | Sticky Note                   | Instructions on uploading files to InfraNodus graph |                            |                          | Step 1 instructions including InfraNodus API key and usage links                                              |
| Sticky Note5                   | Sticky Note                   | Notes about optional better PDF conversion   |                            |                          | Optional: Better PDF Conversion explanation with ConvertAPI link                                              |
| Sticky Note1                   | Sticky Note                   | InfraNodus knowledge graph example image     |                            |                          | InfraNodus knowledge graph example visualization                                                              |
| When chat message received      | Langchain Chat Trigger (disabled) | Receives chat messages from users            |                            | AI Agent                 |                                                                                                               |
| AI Agent                      | Langchain Agent               | Combines memory, LM, and knowledge base to generate responses | When chat message received, Simple Memory, OpenAI Chat Model, Knowledge Base GraphRAG |                          |                                                                                                               |
| Simple Memory                 | Langchain Memory Buffer Window | Maintains chat conversation context          |                            | AI Agent (ai_memory)      |                                                                                                               |
| OpenAI Chat Model             | Langchain LM Chat OpenAI       | GPT-4o-mini language model for response generation |                            | AI Agent (ai_languageModel) |                                                                                                               |
| Knowledge Base GraphRAG        | HTTP Request Tool              | Queries InfraNodus graph for knowledge retrieval |                            | AI Agent (ai_tool)        |                                                                                                               |
| Sticky Note                   | Sticky Note                   | Instructions for chat usage and InfraNodus setup |                            |                          | Step 2 instructions including InfraNodus chat setup and API key usage                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**  
   - Name: `Click ‘Test workflow’ to ingest the documents`  
   - Purpose: To initiate the document ingestion process manually.

2. **Add Google Drive Node (fileFolder resource):**  
   - Name: `Search Google Drive`  
   - Credentials: Configure Google Drive OAuth2 with appropriate account.  
   - Set folderId to the target folder where PDFs/MD/Text files are stored (`13tqp0SaI_v4jG1CFLAZo96isx-UBno4v`).  
   - Enable `Return All` and set queryString to "*" to list all files.  
   - Connect from Manual Trigger node.

3. **Add SplitInBatches Node:**  
   - Name: `Loop Over Items`  
   - Default batch size or adjust to preferred size.  
   - Connect from `Search Google Drive`.

4. **Add Google Drive Node (download operation):**  
   - Name: `Retrieve File`  
   - Credentials: Use same Google Drive OAuth2.  
   - Set `fileId` parameter dynamically from `{{$json.id}}` (ID of each file).  
   - Connect from second output of `Loop Over Items`.

5. **Add Switch Node:**  
   - Name: `Switch`  
   - Configure rules based on binary data MIME type (`{{$binary["data"].mimeType}}`):  
     - PDF → `application/pdf`  
     - Text → `text/plain`  
     - Markdown → `text/markdown`  
     - (Others can be added but not needed here)  
   - Connect from `Retrieve File`.

6. **Add ExtractFromFile Nodes for each file type:**  
   - Name: `Extract from PDF` (operation: pdf) → connected to PDF output of Switch.  
   - Name: `Extract from Text File` (operation: text) → connected to Text output of Switch.  
   - Name: `Extract from Markdown` (operation: text) → connected to Markdown output of Switch.

7. **Add Set Node to map PDF text:**  
   - Name: `Map PDF to Text`  
   - Set field `data` to `{{$json.text}}`.  
   - Connect from `Extract from PDF`.

8. **Add HTTP Request Node to save to InfraNodus graph:**  
   - Name: `InfraNodus Save to Graph`  
   - Method: POST  
   - URL: `https://infranodus.com/api/v1/graphAndStatements?doNotSave=false&includeGraphSummary=true`  
   - Authentication: HTTP Bearer Auth with InfraNodus API key credential.  
   - Body Parameters:  
     - `name`: `"graphrag_from_google_drive"` (graph name)  
     - `text`: `{{$json.data}}` (extracted text)  
     - `=categories`: `["filename: {{$json.name}}"]`  
     - `contextSettings`: `{"squareBracketsProcessing":"IGNORE_BRACKETS"}` (JSON string)  
   - Connect outputs from:  
     - `Map PDF to Text`  
     - `Extract from Text File`  
     - `Extract from Markdown`  
   - Connect output back to second output of `Loop Over Items` for batching continuation.

9. **(Optional) Add HTTP Request Node for ConvertAPI PDF to Text conversion:**  
   - Name: `Convert File to PDF`  
   - Configure multipart form data with binary file input field "data".  
   - Authentication: HTTP Bearer Auth with ConvertAPI credential.  
   - Disabled by default; can be enabled to replace `Map PDF to Text` for better PDF text quality.

10. **(Optional) Add Sticky Notes** for user instructions and documentation.

---

11. **For Interactive Chat Setup (Optional, disabled by default):**

12. **Add Langchain Chat Trigger Node:**  
    - Name: `When chat message received`  
    - Configure webhook and make public.  
    - Initial messages prompt user to ask questions about PDFs or write `/question`.  
    - Disabled initially.

13. **Add Langchain Agent Node:**  
    - Name: `AI Agent`  
    - Configure system message instructing to use the knowledge base for answers.  
    - Connect input from Chat Trigger, memory, language model, and knowledge base tool.

14. **Add Langchain Memory Node:**  
    - Name: `Simple Memory`  
    - Connect output to AI Agent’s memory input.

15. **Add Langchain OpenAI Chat Model Node:**  
    - Name: `OpenAI Chat Model`  
    - Model: `gpt-4o-mini` or preferred GPT model.  
    - Credentials: OpenAI API key configured.  
    - Connect output to AI Agent’s language model input.

16. **Add HTTP Request Tool Node for InfraNodus querying:**  
    - Name: `Knowledge Base GraphRAG`  
    - POST to `https://infranodus.com/api/v1/graphAndAdvice` with parameters:  
      - `name`: `"graphrag_from_google_drive"` (same graph name)  
      - `requestMode`: `"response"`  
      - `prompt`: user chat query (dynamic)  
      - `aiTopics`: `true`  
    - Authentication: InfraNodus API key.  
    - Connect output to AI Agent’s tool input.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Step 1: Upload your PDF / MD / Text files to InfraNodus GraphRAG via Google Drive. Requires InfraNodus account and API key at https://infranodus.com/api-access. Use this to build your knowledge graph.                                   | InfraNodus API access page: https://infranodus.com/api-access                                     |
| Optional: Use ConvertAPI for higher quality PDF-to-text conversion that respects paragraph layout. Requires ConvertAPI account https://convertapi.com?ref=4l54n. Can replace standard PDF extraction in ingestion step.                       | ConvertAPI: https://convertapi.com?ref=4l54n                                                      |
| Step 2: Chat with uploaded files using InfraNodus GraphRAG knowledge base, avoiding vector stores. Activate chat trigger and provide InfraNodus API key. Use consistent graph name for both ingestion and querying.                         | InfraNodus knowledge graph editor example: https://infranodus.com/your_user_name/your_graph_name/edit |
| InfraNodus visualizes text as a knowledge graph highlighting topics and relations, improving retrieval and context understanding beyond vector embeddings.                                                                               | InfraNodus homepage: https://infranodus.com                                                       |
| For multi-knowledge base setups, describe each graph well in InfraNodus project notes to help the AI agent distinguish contexts.                                                                                                         | InfraNodus project notes for RAG enhancement                                                     |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---