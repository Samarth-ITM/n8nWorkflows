Extract Context from Voice Notes with OpenRouter AI & Milvus for RAG Systems

https://n8nworkflows.xyz/workflows/extract-context-from-voice-notes-with-openrouter-ai---milvus-for-rag-systems-7430


# Extract Context from Voice Notes with OpenRouter AI & Milvus for RAG Systems

### 1. Workflow Overview

This workflow, titled **"Context Ingestion Pipeline"**, is designed to process voice note transcriptions and extract structured context data for use in Retrieval-Augmented Generation (RAG) systems. The key use case is to convert raw speech-to-text data (which may contain transcription errors or imprecisions) into precise, reformulated context facts that can be embedded into a vector database (Milvus) for later AI-driven retrieval and inference.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receive and parse incoming webhook POST requests containing voice note data.
- **1.2 Data Preparation:** Extract and normalize relevant fields (title, transcript, timestamp) from the raw webhook data.
- **1.3 AI Context Extraction:** Use an AI agent configured with OpenRouter to clean, reformulate, and extract context-relevant facts from the transcript.
- **1.4 Output Parsing and Formatting:** Parse the AI agent’s output to produce clean, structured plain text facts.
- **1.5 Data Packaging:** Prepare the extracted context data along with metadata (e.g., timestamp) into a text file format.
- **1.6 Vector Embedding and Storage:** Embed the context data using OpenAI embeddings and store it in a Milvus vector database collection for RAG usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Accepts HTTP POST requests from an external source (likely voicenotes.com or a similar system) containing voice note transcription data.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: HTTP Webhook (n8n-nodes-base.webhook)  
    - Configuration: Listens to POST requests on a specified path (`webhook-uuid-placeholder`).  
    - Key expressions: None, it directly receives JSON payload with nested fields for the voice note (`body.data.title`, `body.data.transcript`, `body.timestamp`).  
    - Input: External HTTP POST request  
    - Output: JSON object with voice note metadata and transcript.  
    - Edge cases: Invalid or malformed HTTP requests, missing fields in JSON payload, authorization errors if security is implemented externally.

#### 2.2 Data Preparation

- **Overview:**  
  Extracts specific fields (title, transcript, timestamp) from the webhook payload and normalizes them into top-level workflow variables.

- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**  
  - **Edit Fields**  
    - Type: Set (n8n-nodes-base.set)  
    - Configuration: Assigns three string fields: `title`, `transcript`, and `timestamp` from nested JSON paths in the webhook data.  
      - `title` = `$json.body.data.title`  
      - `transcript` = `$json.body.data.transcript`  
      - `timestamp` = `$json.body.timestamp`  
    - Input: JSON from Webhook  
    - Output: JSON with simplified structure containing only the three fields.  
    - Edge cases: Missing or empty transcript/title/timestamp fields.

#### 2.3 AI Context Extraction

- **Overview:**  
  Processes the raw transcript text using a LangChain AI Agent powered by OpenRouter’s chat model. The agent reformulates the transcript into third-person, corrects transcription errors, and extracts only significant, factual context data formatted as plain text.

- **Nodes Involved:**  
  - AI Agent  
  - OpenRouter Chat Model  
  - Structured Output Parser

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent (@n8n/n8n-nodes-langchain.agent)  
    - Configuration:  
      - Input text: Transcript string from `Edit Fields` node.  
      - System message prompt defines an extensive role: detect mistranscriptions, reformulate first-person to third-person, extract significant facts only, format as plain text under all-caps headers, omit filler and narrative.  
      - Output parser enabled (structured output expected).  
    - Inputs: Transcript text  
    - Outputs: Raw AI-generated context extraction text  
    - Edge cases: AI output might be incomplete, contain unexpected formatting, or fail due to API errors or latency.

  - **OpenRouter Chat Model**  
    - Type: Language Model (@n8n/n8n-nodes-langchain.lmChatOpenRouter)  
    - Configuration: Uses OpenRouter API credentials for chat completions. Passes parameters transparently to AI Agent.  
    - Edge cases: API authentication failures, rate limits, or network errors.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser (@n8n/n8n-nodes-langchain.outputParserStructured)  
    - Configuration: Defines expected JSON schema example for parsing AI output into structured JSON with a single property `output` containing the extracted facts.  
    - Edge cases: Parsing errors if AI output deviates from expected format.

#### 2.4 Output Parsing and Formatting

- **Overview:**  
  Prepares the final output text to include the timestamp and extracted context facts, combining them into a single string for file conversion.

- **Nodes Involved:**  
  - Edit Fields1

- **Node Details:**  
  - **Edit Fields1**  
    - Type: Set (n8n-nodes-base.set)  
    - Configuration:  
      - Creates two fields:  
        - `tite.` (likely a typo for `title`) set from the original title extracted earlier.  
        - `output` field which includes a header with the timestamp and the extracted context facts from AI output.  
      - Output format example:  
        ```
        Context data created: <timestamp>

        CONTEXT:

        <extracted facts>
        ```  
    - Input: AI Agent output and original title  
    - Output: JSON with combined output string ready for file conversion  
    - Edge cases: Typo in field name `tite.` may cause issues downstream.

#### 2.5 Data Packaging

- **Overview:**  
  Converts the combined context text into a text file format suitable for embedding and storage.

- **Nodes Involved:**  
  - Convert to File

- **Node Details:**  
  - **Convert to File**  
    - Type: Convert To File (n8n-nodes-base.convertToFile)  
    - Configuration:  
      - Operation: Convert text property `output` to a text file.  
      - Filename: Dynamically set from `tite.[empty string key]` (which may be an error or misconfiguration).  
    - Input: JSON containing `output` text and `tite.` field  
    - Output: Binary file data with context text  
    - Edge cases: Filename extraction likely broken due to `tite[\"\"]` reference, may cause empty or invalid filenames.

#### 2.6 Vector Embedding and Storage

- **Overview:**  
  Embeds the text file contents as vectors using OpenAI embeddings and stores them in a Milvus vector database collection for efficient retrieval in RAG pipelines.

- **Nodes Involved:**  
  - Embeddings OpenAI  
  - Milvus Vector Store  
  - Default Data Loader

- **Node Details:**  
  - **Embeddings OpenAI**  
    - Type: LangChain Embeddings (@n8n/n8n-nodes-langchain.embeddingsOpenAi)  
    - Configuration: Uses OpenAI API credentials to generate vector embeddings from text.  
    - Input: Not connected directly in the main flow but linked to Milvus node as an embedding source.  
    - Edge cases: API key issues, quota limits, or network errors.

  - **Default Data Loader**  
    - Type: LangChain Document Default Data Loader (@n8n/n8n-nodes-langchain.documentDefaultDataLoader)  
    - Configuration: Handles binary data input (the converted text file).  
    - Input: Binary text file from Convert to File node  
    - Edge cases: Data format mismatches or failure to load document.

  - **Milvus Vector Store**  
    - Type: LangChain Vector Store (@n8n/n8n-nodes-langchain.vectorStoreMilvus)  
    - Configuration:  
      - Mode: Insert new vectors without clearing the collection.  
      - Collection: `user-context-collection` (hosted Milvus instance).  
      - Credentials: Milvus API credentials supplied.  
    - Inputs: Accepts embedding vectors and document loader outputs to insert into the vector database.  
    - Edge cases: Connection failures, credential misconfiguration, collection access issues.

---

### 3. Summary Table

| Node Name               | Node Type                                   | Functional Role                     | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                                                                         |
|-------------------------|---------------------------------------------|-----------------------------------|----------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                 | HTTP Webhook                                | Receive voice note data via POST  | External HTTP POST    | Edit Fields              | ## Context data  Voicenotes.com tag trigger for 'conext data'                                                                                      |
| Edit Fields             | Set                                         | Extract and normalize fields      | Webhook              | AI Agent                 | ## Narrow fields                                                                                                                                     |
| AI Agent                | LangChain Agent                             | Extract context facts from transcript | Edit Fields           | Edit Fields1             | ## Context data extraction agent  Parses transcript and isolates context rich text                                                                 |
| OpenRouter Chat Model   | LangChain LM Chat Model                     | AI language model for agent       | AI Agent (ai_languageModel) | AI Agent              |                                                                                                                                                     |
| Structured Output Parser | LangChain Output Parser                      | Parse AI output into JSON         | OpenRouter Chat Model | AI Agent (ai_outputParser) |                                                                                                                                                     |
| Edit Fields1            | Set                                         | Format final output with timestamp and context | AI Agent          | Convert to File          | ## Context data prepared for embedding  Timestamp injected into agent output                                                                       |
| Convert to File         | Convert To File                             | Convert context text to file      | Edit Fields1          | Milvus Vector Store      |                                                                                                                                                     |
| Milvus Vector Store     | LangChain Vector Store                      | Store embeddings in vector db     | Convert To File, Embeddings OpenAI, Default Data Loader | -                      | ## Embedding  Context data embedded into Milvus vector database (hosted)                                                                           |
| Embeddings OpenAI       | LangChain Embeddings                        | Generate vector embeddings        | -                    | Milvus Vector Store      |                                                                                                                                                     |
| Default Data Loader     | LangChain Document Data Loader              | Load text file as document        | -                    | Milvus Vector Store      |                                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node**  
   - Type: HTTP Webhook  
   - Method: POST  
   - Path: e.g., `webhook-uuid-placeholder`  
   - Purpose: Receive JSON payload with voice note data.

2. **Create Set node named "Edit Fields"**  
   - Type: Set  
   - Fields to set:  
     - `title` = Expression: `$json.body.data.title`  
     - `transcript` = Expression: `$json.body.data.transcript`  
     - `timestamp` = Expression: `$json.body.timestamp`  
   - Connect Webhook → Edit Fields.

3. **Create LangChain Chat Model node named "OpenRouter Chat Model"**  
   - Type: LangChain LM Chat Model  
   - Credentials: Configure OpenRouter API credentials  
   - No specific prompt here; this node is linked to AI Agent.

4. **Create LangChain Output Parser node named "Structured Output Parser"**  
   - Type: LangChain Output Parser  
   - JSON Schema Example (copy from example):  
     ```json
     {
       "output": "User moved to a new city recently.\nUser likes pizza.\nUser's favorite type of pizza is Margarita.\nUser's apartment has a view of the downtown area."
     }
     ```
   - This parser will expect AI output in this format.

5. **Create LangChain Agent node named "AI Agent"**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text input: Expression: `$json.transcript` from Edit Fields node  
     - System Message: (Copy the detailed instructions for context extraction from the original workflow, emphasizing reformulation, error correction, third-person conversion, fact extraction, and formatting rules.)  
     - Enable output parser, and select the Structured Output Parser node.  
   - Link OpenRouter Chat Model as `ai_languageModel` input for AI Agent.  
   - Link Structured Output Parser as `ai_outputParser` input for AI Agent.  
   - Connect Edit Fields → AI Agent.

6. **Create Set node named "Edit Fields1"**  
   - Type: Set  
   - Fields to set:  
     - `tite.` = Expression: `$node["Edit Fields"].json["title"]` (fix typo to `title` if desired)  
     - `output` = Expression:  
       ```
       Context data created: {{$node["Webhook"].json["body"]["timestamp"]}}

       CONTEXT:

       {{$json["output"]}}
       ```  
   - Connect AI Agent → Edit Fields1.

7. **Create Convert To File node named "Convert to File"**  
   - Type: Convert To File  
   - Operation: toText  
   - Source Property: `output` (from Edit Fields1)  
   - File Name: Expression based on title, e.g., `$json["title"]` (fix to avoid empty string key)  
   - Connect Edit Fields1 → Convert to File.

8. **Create LangChain Document Default Data Loader node named "Default Data Loader"**  
   - Type: Document Default Data Loader  
   - Data Type: binary  
   - Connect Convert to File → Default Data Loader as binary input.

9. **Create LangChain Embeddings OpenAI node named "Embeddings OpenAI"**  
   - Type: Embeddings OpenAI  
   - Credentials: Configure OpenAI API credentials.

10. **Create LangChain Vector Store node named "Milvus Vector Store"**  
    - Type: Vector Store Milvus  
    - Credentials: Configure Milvus API credentials.  
    - Mode: Insert (no clearing)  
    - Collection: `user-context-collection`  
    - Connect Convert to File main output → Milvus Vector Store main input  
    - Connect Embeddings OpenAI `ai_embedding` output → Milvus Vector Store `ai_embedding` input  
    - Connect Default Data Loader `ai_document` output → Milvus Vector Store `ai_document` input

11. **Connect nodes in order:**

    - Webhook → Edit Fields  
    - Edit Fields → AI Agent  
    - AI Agent → Edit Fields1  
    - Edit Fields1 → Convert to File  
    - Convert to File → Default Data Loader  
    - Embeddings OpenAI & Default Data Loader → Milvus Vector Store

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The system message for the AI Agent is critical to ensure proper reformulation and extraction of context facts from noisy speech-to-text data. | Embedded in the AI Agent node configuration.                                                   |
| The workflow relies on OpenRouter for chat completions and OpenAI for embeddings, requiring valid API credentials for both services.           | Credentials must be set up in n8n prior to execution.                                          |
| Milvus vector store is used for persistent storage of embeddings, enabling fast similarity search for RAG systems.                             | Milvus API credentials and collection configuration required.                                  |
| Sticky notes in the workflow provide helpful context on node groupings and functional roles, such as the emphasis on 'Narrow fields' and 'Embedding'. | Sticky Notes nodes positioned near relevant nodes (Webhook, Edit Fields, AI Agent, etc.).      |
| Potential typo in "Edit Fields1" node field name `tite.` should be reviewed and corrected to `title` for consistency and filename generation. | May cause issues in filename or metadata downstream if left uncorrected.                       |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated n8n workflow. The processing respects content policies and contains no illegal or protected data. All handled data is legal and public.