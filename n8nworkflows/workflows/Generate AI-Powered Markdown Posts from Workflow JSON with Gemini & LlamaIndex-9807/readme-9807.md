Generate AI-Powered Markdown Posts from Workflow JSON with Gemini & LlamaIndex

https://n8nworkflows.xyz/workflows/generate-ai-powered-markdown-posts-from-workflow-json-with-gemini---llamaindex-9807


# Generate AI-Powered Markdown Posts from Workflow JSON with Gemini & LlamaIndex

### 1. Workflow Overview

This workflow is designed to automate the generation of AI-powered, SEO-optimized markdown posts from technical JSON workflow data. It is intended for content creators, developers, and marketers who want to transform raw n8n workflow JSON files into polished markdown posts suitable for publication on the n8n community platform or blogs. The workflow incorporates advanced AI models (Google Gemini, Cohere reranker, LlamaIndex) and integrates with Google Drive for document updates.

The workflow is logically grouped into these main blocks:

- **1.1 Input Reception & Extraction:** Receives JSON workflow files via a form submission and extracts JSON content.
- **1.2 AI Knowledge Base Preparation:** Downloads and parses a Google Drive document representing the knowledge base, updating an in-memory vector store.
- **1.3 AI Content Generation:** Uses advanced large language models (Google Gemini), embeddings (Google Gemini embeddings), and a reranker (Cohere) to generate optimized markdown posts based on the JSON input and knowledge base.
- **1.4 Workflow Monitoring & Parsing:** Monitors asynchronous document parsing jobs via the LlamaIndex API.
- **1.5 Auxiliary & Documentation:** Sticky notes for user guidance and credentials setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Extraction

**Overview:**  
This block handles the initial reception of the workflow JSON file from the user via a form and extracts the JSON content for further processing.

**Nodes Involved:**  
- On form submission  
- Extract from File  
- n8ncreator  

**Node Details:**  

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point for the workflow; triggers execution when a user submits a form with a JSON file upload.  
  - Configuration: Accepts only `.json` files with one required field labeled "Input Json Workflow".  
  - Inputs: External webhook trigger via form submission.  
  - Outputs: Passes binary file data to "Extract from File".  
  - Failure Modes: Form not submitted, invalid file type, empty upload.  
  - Version: 2.2  

- **Extract from File**  
  - Type: Extract from File  
  - Role: Converts the uploaded JSON file binary data into usable JSON data stored in the workflow context under the key "workflow".  
  - Configuration: Operation set to "fromJson", input binary property name matches form field "Input_Json_Workflow".  
  - Inputs: Binary file data from form.  
  - Outputs: JSON content extracted from file, forwarded to "n8ncreator".  
  - Failure Modes: Malformed JSON file, extraction errors.  
  - Version: 1  

- **n8ncreator**  
  - Type: LangChain Agent  
  - Role: Core AI processing node that transforms the input JSON workflow into a publication-ready markdown post. It uses a detailed system prompt tailored for SEO and content quality.  
  - Configuration:  
    - Input text: The extracted JSON stored under `workflow`.  
    - System message: Detailed instructions for generating SEO-optimized markdown posts with knowledge base consultation.  
  - Inputs: JSON workflow data.  
  - Outputs: Generated markdown content.  
  - Failure Modes: AI model downtime, prompt execution errors, API quota limits.  
  - Version: 2.1  

---

#### 2.2 AI Knowledge Base Preparation

**Overview:**  
This block monitors a Google Drive document used as a knowledge base, downloads it on updates, and parses it using LlamaIndex to update an in-memory vector store used by the AI for content generation.

**Nodes Involved:**  
- Knowledge Base Updated Trigger  
- Download Knowledge Document  
- Parse Document via LlamaIndex  
- Monitor Document Processing  
- Check Parsing Completion  
- Wait Before Status Recheck  
- Retrieve Parsed Content  
- N8N KB  
- Data Loader  

**Node Details:**  

- **Knowledge Base Updated Trigger**  
  - Type: Google Drive Trigger  
  - Role: Watches a specific Google Drive document for updates, polling every minute.  
  - Configuration: Watches a document by file ID (Google Docs link cached).  
  - Inputs: Google Drive file update event.  
  - Outputs: Triggers the download.  
  - Failure Modes: OAuth token expiration, API quota limits.  
  - Version: 1  

- **Download Knowledge Document**  
  - Type: Google Drive  
  - Role: Downloads the updated Google Docs file content as binary to be parsed.  
  - Configuration: Downloads file by ID using OAuth2 credentials.  
  - Inputs: Trigger from Drive trigger.  
  - Outputs: Binary document data.  
  - Failure Modes: File access denied, download failures.  
  - Version: 3  

- **Parse Document via LlamaIndex**  
  - Type: HTTP Request  
  - Role: Uploads the downloaded document to LlamaIndex API for parsing into structured data.  
  - Configuration: POST multipart/form-data with binary file, uses HTTP header authentication via stored credential.  
  - Inputs: Binary document data.  
  - Outputs: Job ID for parsing process.  
  - Failure Modes: API errors, authentication failure, rate limits.  
  - Version: 4.2  

- **Monitor Document Processing**  
  - Type: HTTP Request  
  - Role: Polls LlamaIndex API to check parsing job status using job ID.  
  - Configuration: GET request with job ID in URL, authenticated via HTTP header.  
  - Inputs: Job ID from parsing node or wait node.  
  - Outputs: JSON status of job.  
  - Failure Modes: API downtime, network errors.  
  - Version: 4.2  

- **Check Parsing Completion**  
  - Type: If  
  - Role: Checks if parsing status equals "SUCCESS". Branches workflow accordingly.  
  - Configuration: Condition checks `$json.status === "SUCCESS"`.  
  - Inputs: Status JSON.  
  - Outputs: Success branch triggers content retrieval; else triggers wait.  
  - Failure Modes: JSON path errors, unexpected status values.  
  - Version: 2.2  

- **Wait Before Status Recheck**  
  - Type: Wait  
  - Role: Delays the next status check by 10 seconds.  
  - Configuration: Fixed 10 seconds delay.  
  - Inputs: Failure branch of parsing completion.  
  - Outputs: Retriggers monitor node.  
  - Failure Modes: None significant.  
  - Version: 1.1  

- **Retrieve Parsed Content**  
  - Type: HTTP Request  
  - Role: Retrieves parsed document content in markdown format from LlamaIndex once parsing is complete.  
  - Configuration: GET request to job result URL with markdown accept header.  
  - Inputs: Job ID via JSON.  
  - Outputs: Parsed markdown content JSON.  
  - Failure Modes: API errors, content retrieval failure.  
  - Version: 4.2  

- **N8N KB**  
  - Type: Vector Store In-Memory (LangChain)  
  - Role: Inserts or retrieves documents from an in-memory vector database keyed as "n8n KB". Used by AI to answer questions or enrich content.  
  - Configuration: Mode is "insert" for updates; "retrieve-as-tool" for question answering.  
  - Inputs: Markdown content from parsed documents (insert mode).  
  - Outputs: Enriched knowledge base accessible by AI.  
  - Failure Modes: Memory overflow, indexing errors.  
  - Version: 1.2  

- **Data Loader**  
  - Type: Document Default Data Loader (LangChain)  
  - Role: Loads documents into the vector store in a default manner, supporting AI embedding insertion.  
  - Configuration: Default options.  
  - Inputs: Markdown content.  
  - Outputs: Structured data to N8N KB node.  
  - Failure Modes: Data format incompatibility.  
  - Version: 1.1  

---

#### 2.3 AI Content Generation

**Overview:**  
This block applies AI models and embeddings to generate SEO-optimized markdown posts using the input JSON workflow and knowledge base.

**Nodes Involved:**  
- GEmini 2.5 pro  
- Embedd 004  
- Reranker Cohere  
- n8n kb (in "retrieve-as-tool" mode)  
- n8ncreator  
- N8N KB (in "insert" mode)  

**Node Details:**  

- **GEmini 2.5 pro**  
  - Type: LM Chat (Google Gemini)  
  - Role: Primary large language model generating natural language content.  
  - Configuration: Model set to "models/gemini-2.5-pro".  
  - Inputs: Text prompts, possibly system messages and user input.  
  - Outputs: Generated text responses.  
  - Failure Modes: API limits, model latency, request failures.  
  - Version: 1  

- **Embedd 004**  
  - Type: Embeddings (Google Gemini)  
  - Role: Generates vector embeddings from text used for semantic search and knowledge base insertion.  
  - Configuration: Default parameters with Google Palm API credentials.  
  - Inputs: Text content to embed.  
  - Outputs: Embedding vectors.  
  - Failure Modes: API errors, embedding quality issues.  
  - Version: 1  

- **Reranker Cohere**  
  - Type: Reranker (Cohere)  
  - Role: Re-ranks retrieved documents or responses based on relevance scores to improve AI answer quality.  
  - Configuration: Uses Cohere API credentials.  
  - Inputs: Candidate documents or contents to rerank.  
  - Outputs: Ranked list for better selection.  
  - Failure Modes: API downtime, ranking inconsistencies.  
  - Version: 1  

- **n8n kb**  
  - Type: Vector Store In-Memory (LangChain)  
  - Role: Operates in two modes: "retrieve-as-tool" to provide knowledge base answers, and "insert" to update KB.  
  - Configuration: Uses memory key "n8n KB" and enables reranker integration.  
  - Inputs: Text queries or documents for insertion.  
  - Outputs: Retrieved knowledge snippets or updated KB state.  
  - Failure Modes: Memory capacity, retrieval accuracy.  
  - Version: 1.2  

---

#### 2.4 Auxiliary & Documentation

**Overview:**  
This block contains sticky notes used for user guidance, credential setup instructions, and appreciation messages.

**Nodes Involved:**  
- Sticky Note2 (Knowledge Base)  
- Sticky Note3 (Credentials Setup Guide)  
- Sticky Note4 (Support Creator)  

**Node Details:**  

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Displays "Knowledge Base" header for visual grouping.  
  - Configuration: Large yellow note for section labeling.  
  - Inputs/Outputs: None.  
  - Version: 1  

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Detailed credential setup guide for Azure OpenAI, LlamaIndex API, and Google Drive OAuth2.  
  - Configuration: Extensive textual instructions with links.  
  - Inputs/Outputs: None.  
  - Version: 1  

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Encourages users to support the creator via PayPal with embedded link.  
  - Configuration: Small note with formatted markdown content.  
  - Inputs/Outputs: None.  
  - Version: 1  

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                                  | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                          |
|----------------------------|--------------------------------------|-------------------------------------------------|----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger                         | Receives JSON file upload via form               | -                          | Extract from File               |                                                                                                    |
| Extract from File          | Extract from File                    | Extracts JSON content from uploaded file         | On form submission          | n8ncreator                    |                                                                                                    |
| n8ncreator                | LangChain Agent                     | Transforms JSON workflow into SEO markdown post  | Extract from File, n8n kb, GEmini 2.5 pro | -                           |                                                                                                    |
| Knowledge Base Updated Trigger | Google Drive Trigger               | Watches Google Docs file for updates              | -                          | Download Knowledge Document    |                                                                                                    |
| Download Knowledge Document | Google Drive                       | Downloads updated Google Docs file                 | Knowledge Base Updated Trigger | Parse Document via LlamaIndex |                                                                                                    |
| Parse Document via LlamaIndex | HTTP Request                     | Uploads document to cloud parsing service         | Download Knowledge Document | Monitor Document Processing    |                                                                                                    |
| Monitor Document Processing | HTTP Request                      | Polls parsing job status                           | Parse Document via LlamaIndex, Wait Before Status Recheck | Check Parsing Completion |                                                                                                    |
| Check Parsing Completion    | If                                 | Checks if parsing finished successfully            | Monitor Document Processing | Retrieve Parsed Content, Wait Before Status Recheck |                                                                                                    |
| Wait Before Status Recheck  | Wait                               | Delays polling retry by 10 seconds                 | Check Parsing Completion (failure) | Monitor Document Processing |                                                                                                    |
| Retrieve Parsed Content     | HTTP Request                      | Retrieves parsed markdown content                   | Check Parsing Completion    | N8N KB                        |                                                                                                    |
| N8N KB                     | Vector Store In-Memory (LangChain) | Inserts/retrieves documents for AI knowledge base | Retrieve Parsed Content, Embedd 004, Data Loader | n8n kb (tool retrieval), n8ncreator |                                                                                                    |
| Data Loader                | Document Default Data Loader        | Loads documents into vector store                   | Retrieve Parsed Content     | N8N KB                        |                                                                                                    |
| n8n kb                     | Vector Store In-Memory (LangChain) | Provides knowledge base answers to AI agent        | Embedd 004, Reranker Cohere | n8ncreator                    |                                                                                                    |
| Embedd 004                 | Embeddings (Google Gemini)          | Creates text embeddings for vector store           | -                          | n8n kb, N8N KB                |                                                                                                    |
| Reranker Cohere            | Reranker (Cohere)                   | Re-ranks documents for relevance                    | -                          | n8n kb                       |                                                                                                    |
| GEmini 2.5 pro             | LM Chat (Google Gemini)             | Generates natural language content                  | -                          | n8ncreator                    |                                                                                                    |
| Sticky Note2               | Sticky Note                        | Visual header for Knowledge Base section            | -                          | -                             |                                                                                                    |
| Sticky Note3               | Sticky Note                        | Credential setup instructions                        | -                          | -                             |                                                                                                    |
| Sticky Note4               | Sticky Note                        | Support creator message                              | -                          | -                             | ## ☕ Appreciate This Workflow? Support the creator by sending coffee: **PayPal:** https://paypal.me/khmuhtadin Thank you! 🚀 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named `On form submission`.**  
   - Type: Form Trigger (version 2.2)  
   - Configure form title as "Test Json".  
   - Add one required field: File type, label "Input Json Workflow", accept `.json` files only.  
   - This node will generate a webhook URL for form submission.

2. **Add an Extract from File node named `Extract from File`.**  
   - Type: Extract from File (version 1)  
   - Set operation to "fromJson".  
   - Set binary property name to "Input_Json_Workflow" (matches form field).  
   - Connect `On form submission` main output to this node's input.

3. **Add a LangChain Agent node named `n8ncreator`.**  
   - Type: LangChain Agent (version 2.1)  
   - In parameters, set the input text as `{{$json.workflow}}` to pass the extracted JSON.  
   - Paste the detailed system message prompt that instructs on transforming JSON workflow to SEO markdown posts, including knowledge base usage and compliance checks.  
   - Connect the output of `Extract from File` to this node's input.

4. **Create a Google Drive Trigger node named `Knowledge Base Updated Trigger`.**  
   - Type: Google Drive Trigger (version 1)  
   - Configure to poll every minute.  
   - Set to trigger on a specific file by its ID (use your Google Docs file ID).  
   - Configure Google Drive OAuth2 credential.

5. **Add a Google Drive node named `Download Knowledge Document`.**  
   - Type: Google Drive (version 3)  
   - Operation: "Download"  
   - File ID: Same Google Docs file ID as in trigger.  
   - Connect `Knowledge Base Updated Trigger` output to this node.

6. **Add an HTTP Request node named `Parse Document via LlamaIndex`.**  
   - Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`  
   - Content-Type: multipart/form-data  
   - Body parameters: Add a form binary data parameter named `file`, using input binary property "data" from previous node.  
   - Add HTTP header: `accept: application/json`.  
   - Use HTTP Header Auth credential with your LlamaIndex API key.  
   - Connect `Download Knowledge Document` output to this node.

7. **Add an HTTP Request node named `Monitor Document Processing`.**  
   - Type: HTTP Request (version 4.2)  
   - Method: GET  
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{$json.id}}`  
   - Header: `accept: application/json`  
   - Use same HTTP Header Auth credential.  
   - Connect output of `Parse Document via LlamaIndex` and `Wait Before Status Recheck` nodes to this node.

8. **Add an If node named `Check Parsing Completion`.**  
   - Type: If (version 2.2)  
   - Condition: Check if `$json.status` equals "SUCCESS" (case sensitive).  
   - Connect `Monitor Document Processing` output to this node.

9. **Add a Wait node named `Wait Before Status Recheck`.**  
   - Type: Wait (version 1.1)  
   - Set wait time to 10 seconds.  
   - Connect `Check Parsing Completion` node's "false" output to this node.  
   - Connect this node back to `Monitor Document Processing` to poll again.

10. **Add an HTTP Request node named `Retrieve Parsed Content`.**  
    - Type: HTTP Request (version 4.2)  
    - Method: GET  
    - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{$json.id}}/result/markdown`  
    - Header: `accept: application/json`  
    - Use same LlamaIndex HTTP Header Auth.  
    - Connect `Check Parsing Completion` node's "true" output to this node.

11. **Add a Document Default Data Loader node named `Data Loader`.**  
    - Type: Document Default Data Loader (version 1.1)  
    - Use default options.  
    - Connect `Retrieve Parsed Content` output to this node.

12. **Add a Vector Store In-Memory node named `N8N KB`.**  
    - Type: Vector Store In-Memory (version 1.2)  
    - Set mode to "insert".  
    - Set memory key to "n8n KB".  
    - Connect `Data Loader` output to this node.

13. **Add a Vector Store In-Memory node named `n8n kb`.**  
    - Type: Vector Store In-Memory (version 1.2)  
    - Set mode to "retrieve-as-tool".  
    - Memory key: "n8n KB".  
    - Enable reranker usage.  
    - Configure tool name as "knowledge_base".  
    - Connect outputs of `Embedd 004` and `Reranker Cohere` nodes to this node.  
    - Connect this node’s output to `n8ncreator` to provide knowledge base answers.

14. **Add an Embeddings node named `Embedd 004`.**  
    - Type: Embeddings Google Gemini (version 1)  
    - Use Google Palm API credentials.  
    - Connect output to `n8n kb` and `N8N KB`.

15. **Add a Reranker node named `Reranker Cohere`.**  
    - Type: Reranker Cohere (version 1)  
    - Use Cohere API credentials.  
    - Connect output to `n8n kb`.

16. **Add a LM Chat node named `GEmini 2.5 pro`.**  
    - Type: LM Chat Google Gemini (version 1)  
    - Model name: "models/gemini-2.5-pro".  
    - Use Google Palm API credentials.  
    - Connect output to `n8ncreator` for language model processing.

17. **Add Sticky Notes as visual aids:**  
    - Sticky Note2: Label "Knowledge Base" section.  
    - Sticky Note3: Paste detailed credential setup instructions for Azure OpenAI, LlamaIndex, Google Drive OAuth2.  
    - Sticky Note4: Appreciation message with PayPal link.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Support the creator by sending coffee: PayPal link https://paypal.me/khmuhtadin. Thank you! 🚀                                                                                                                                                                                                                               | Sticky Note4 content                                                                                 |
| Credential setup instructions for Azure OpenAI, LlamaIndex API, Google Drive OAuth2 including steps to create keys and configure n8n credentials.                                                                                                                                                                         | Sticky Note3 content                                                                                 |
| Knowledge Base is maintained as a Google Docs document monitored by Google Drive Trigger, parsed with LlamaIndex, and stored in an in-memory vector store to enrich AI content generation.                                                                                                                                  | Sticky Note2 and workflow logic                                                                      |
| Use this workflow in an n8n instance with appropriate credentials for Google Drive OAuth2, LlamaIndex API, Google Gemini API, and Cohere API to ensure full functionality.                                                                                                                                                   | Credentials section and node configurations                                                        |
| The workflow includes detailed instructions in the LangChain agent prompt to ensure SEO optimization, compliance with n8n community content policies, and generation of high-quality markdown posts suitable for the n8n creators’ platform.                                                                                | n8ncreator node system prompt                                                                        |
| The Google Gemini model version “models/gemini-2.5-pro” is used as the primary language model; ensure API access for this model is configured and active.                                                                                                                                                                    | GEmini 2.5 pro node                                                                                  |
| The workflow is designed to handle asynchronous document parsing by polling the parsing job until success before retrieving content, avoiding premature retrieval errors.                                                                                                                                                  | Monitor Document Processing, Wait, and If nodes                                                    |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. All data processed is legal and publicly available. No illegal, offensive, or protected material is included.