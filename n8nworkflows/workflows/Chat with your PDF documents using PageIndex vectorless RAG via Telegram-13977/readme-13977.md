Chat with your PDF documents using PageIndex vectorless RAG via Telegram

https://n8nworkflows.xyz/workflows/chat-with-your-pdf-documents-using-pageindex-vectorless-rag-via-telegram-13977


# Chat with your PDF documents using PageIndex vectorless RAG via Telegram

## 1. Workflow Overview

This workflow implements a **Telegram-based PDF question-answering bot** powered by **PageIndex** using a **vectorless RAG** approach. Instead of embeddings and a vector database, PageIndex indexes uploaded PDFs into a hierarchical structure and uses LLM reasoning over that structure to answer questions.

It contains **two independent flows**, both using the same Telegram bot credentials:

### 1.1 Flow 1 — PDF Knowledge Upload
This branch is triggered when a user sends a document to the Telegram bot. The workflow downloads the file from Telegram and uploads it to PageIndex for indexing.

### 1.2 Flow 2 — Q&A Chat
This branch is triggered when a user sends a text message to the Telegram bot. The workflow retrieves all indexed PageIndex documents, extracts their IDs, sends the user’s question to PageIndex’s chat endpoint, and replies back in Telegram with the generated answer.

### 1.3 Shared Integration Context
Both flows depend on:
- **Telegram Bot API credentials**
- **PageIndex API key**
- Correct routing of Telegram message fields such as:
  - `message.document.file_id`
  - `message.text`
  - `message.from.id`

Because both entry nodes listen to Telegram `message` updates, the workflow assumes that:
- document uploads are intended for Flow 1
- plain text messages are intended for Flow 2

No explicit filtering is implemented in the workflow JSON.

---

## 2. Block-by-Block Analysis

## 2.1 Workflow Summary and Documentation Layer

**Overview:**  
This block does not affect execution. It provides embedded documentation explaining the overall architecture, credentials, and the distinction between upload and Q&A behavior.

**Nodes Involved:**
- Sticky Note - Summary

### Node Details

#### Sticky Note - Summary
- **Type and technical role:** Sticky Note; visual documentation only.
- **Configuration choices:** Contains a high-level summary of the solution, required credentials, and PageIndex’s indexing model.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None; non-executable.
- **Sub-workflow reference:** None.

---

## 2.2 Flow 1 — PDF Upload Entry and Download

**Overview:**  
This block listens for Telegram messages containing a document and retrieves the corresponding file from Telegram storage. It prepares binary PDF data for upload to PageIndex.

**Nodes Involved:**
- Sticky Note - Flow 1 Header
- Sticky Note - Receive PDF
- Sticky Note - Download PDF
- Receive PDF Document
- Download PDF File

### Node Details

#### Sticky Note - Flow 1 Header
- **Type and technical role:** Sticky Note; visual section header.
- **Configuration choices:** Explains that this flow is for one-time PDF indexing.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note - Receive PDF
- **Type and technical role:** Sticky Note; visual node annotation.
- **Configuration choices:** Documents that the expected input is a Telegram message with a PDF document attachment.
- **Key expressions or variables used:** Refers conceptually to `message.document.file_id`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** Warns that non-PDF files may fail later.
- **Sub-workflow reference:** None.

#### Receive PDF Document
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`; entry point for document uploads.
- **Configuration choices:**
  - Watches Telegram `message` updates.
  - Uses Telegram credentials named **Telegram account**.
- **Key expressions or variables used:** No expressions in configuration, but downstream logic expects `message.document.file_id`.
- **Input and output connections:**
  - No incoming nodes.
  - Outgoing to **Download PDF File**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - If the incoming Telegram message is only text and has no `document`, downstream expression access may fail.
  - Because it listens to all `message` updates, it is not restricted to PDFs.
  - Telegram webhook/credential errors can prevent triggering.
- **Sub-workflow reference:** None.

#### Sticky Note - Download PDF
- **Type and technical role:** Sticky Note; visual annotation.
- **Configuration choices:** Describes that the file is fetched from Telegram before temporary storage expires.
- **Key expressions or variables used:** Mentions `message.document.file_id`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** Notes Telegram file availability timing.
- **Sub-workflow reference:** None.

#### Download PDF File
- **Type and technical role:** `n8n-nodes-base.telegram`; Telegram API action node used to fetch a file.
- **Configuration choices:**
  - Resource: **file**
  - File ID expression: `={{ $json.message.document.file_id }}`
  - Uses Telegram credentials named **Telegram account**
- **Key expressions or variables used:**
  - `{{$json.message.document.file_id}}`
- **Input and output connections:**
  - Input from **Receive PDF Document**
  - Output to **Index PDF on PageIndex**
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Fails if `message.document.file_id` is missing.
  - May fail if the document was deleted or Telegram file retrieval expires.
  - If the file is not actually a PDF, PageIndex may reject it later.
  - Credential or permission errors may prevent file download.
- **Sub-workflow reference:** None.

---

## 2.3 Flow 1 — PDF Indexing in PageIndex

**Overview:**  
This block uploads the downloaded PDF binary to the PageIndex API. PageIndex then creates a tree-based document index and returns a document identifier.

**Nodes Involved:**
- Sticky Note - Index PDF
- Index PDF on PageIndex

### Node Details

#### Sticky Note - Index PDF
- **Type and technical role:** Sticky Note; visual annotation.
- **Configuration choices:** Explains expected binary input and `doc_id` output, and mentions processing duration.
- **Key expressions or variables used:** Refers conceptually to binary PDF data and returned `doc_id`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** Notes that processing may take 10–60 seconds depending on PDF size.
- **Sub-workflow reference:** None.

#### Index PDF on PageIndex
- **Type and technical role:** `n8n-nodes-base.httpRequest`; multipart file upload to PageIndex.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.pageindex.ai/doc/`
  - Content type: `multipart-form-data`
  - Sends body: yes
  - Body includes binary form field:
    - name: `file`
    - source binary property: `data`
  - Sends header:
    - `api_key: YOUR_PAGEINDEX_API_KEY`
- **Key expressions or variables used:** No dynamic expression is used in the configured parameters; it relies on incoming binary property `data`.
- **Input and output connections:**
  - Input from **Download PDF File**
  - No downstream output connection
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Hardcoded placeholder API key must be replaced.
  - Request will fail if binary property `data` is missing or renamed.
  - Large files may trigger timeout or API-side file-size limits.
  - Invalid or corrupted PDF files may be rejected by PageIndex.
  - Successful upload does not guarantee immediate query readiness if indexing is asynchronous.
- **Sub-workflow reference:** None.

---

## 2.4 Flow 2 — Question Intake

**Overview:**  
This block receives Telegram text messages that are treated as user questions. It serves as the entry point for the Q&A path.

**Nodes Involved:**
- Sticky Note - Flow 2 Header
- Sticky Note - Receive Question
- Receive User Question

### Node Details

#### Sticky Note - Flow 2 Header
- **Type and technical role:** Sticky Note; visual section header.
- **Configuration choices:** Explains that any text message is treated as a question and answered through PageIndex.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note - Receive Question
- **Type and technical role:** Sticky Note; visual annotation.
- **Configuration choices:** Documents that the node expects text input and exposes question text plus sender ID.
- **Key expressions or variables used:** References `message.text` and `message.from.id`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** Notes that all text messages trigger the flow.
- **Sub-workflow reference:** None.

#### Receive User Question
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`; Telegram message trigger for Q&A.
- **Configuration choices:**
  - Watches Telegram `message` updates.
  - Uses Telegram credentials named **Telegram account**.
- **Key expressions or variables used:** No direct expressions configured; downstream logic uses `message.text` and `message.from.id`.
- **Input and output connections:**
  - No incoming nodes.
  - Outgoing to **Fetch All Indexed Documents**
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Because it also listens to all `message` updates, document messages may trigger it as well.
  - If `message.text` is absent, later expressions may fail.
  - Telegram webhook conflicts can arise if multiple workflows use the same bot improperly.
- **Sub-workflow reference:** None.

---

## 2.5 Flow 2 — Document Retrieval and ID Extraction

**Overview:**  
This block pulls the list of documents indexed in the PageIndex account and converts that list into a simplified array of document IDs to be passed into the chat endpoint.

**Nodes Involved:**
- Sticky Note - Fetch Docs
- Sticky Note - Extract IDs
- Fetch All Indexed Documents
- Extract Document IDs

### Node Details

#### Sticky Note - Fetch Docs
- **Type and technical role:** Sticky Note; visual annotation.
- **Configuration choices:** Explains that this step retrieves all indexed documents and that only completed ones are query-ready.
- **Key expressions or variables used:** Refers conceptually to `status`, `id`, `name`, and `pageNum`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** Notes that not all listed docs may be completed.
- **Sub-workflow reference:** None.

#### Fetch All Indexed Documents
- **Type and technical role:** `n8n-nodes-base.httpRequest`; GET request to list PageIndex documents.
- **Configuration choices:**
  - URL: `https://api.pageindex.ai/docs`
  - Default method: GET
  - Sends header:
    - `api_key: YOUR_PAGEINDEX_API_KEY`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from **Receive User Question**
  - Output to **Extract Document IDs**
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Hardcoded placeholder API key must be replaced.
  - API auth failures will stop the flow.
  - If the response shape changes and no longer returns `documents`, the next node breaks.
  - The workflow does not filter out incomplete documents even though the sticky note says only `completed` docs are ready.
- **Sub-workflow reference:** None.

#### Sticky Note - Extract IDs
- **Type and technical role:** Sticky Note; visual annotation.
- **Configuration choices:** Explains that the array of document objects is transformed into `doc_ids`.
- **Key expressions or variables used:** Refers conceptually to `documents.map(d => d.id)`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** Passing many doc IDs may enlarge the query scope and affect performance or answer relevance.
- **Sub-workflow reference:** None.

#### Extract Document IDs
- **Type and technical role:** `n8n-nodes-base.set`; creates a new field containing the array of PageIndex document IDs.
- **Configuration choices:**
  - Adds assignment:
    - `doc_ids` as type `array`
    - value: `={{ $json.documents.map(d => d.id) }}`
- **Key expressions or variables used:**
  - `{{$json.documents.map(d => d.id)}}`
- **Input and output connections:**
  - Input from **Fetch All Indexed Documents**
  - Output to **LLM Reasoning over Document Tree**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Fails if `documents` is undefined or not an array.
  - Empty array is possible if no documents are indexed.
  - No filtering on document status means incomplete or failed docs may be included.
- **Sub-workflow reference:** None.

---

## 2.6 Flow 2 — PageIndex Chat Reasoning and Telegram Reply

**Overview:**  
This block sends the user question and the array of PageIndex document IDs to the PageIndex chat completion API, then sends the generated answer back to the originating Telegram user.

**Nodes Involved:**
- Sticky Note - LLM Reasoning
- Sticky Note - Send Answer
- LLM Reasoning over Document Tree
- Send Answer to User

### Node Details

#### Sticky Note - LLM Reasoning
- **Type and technical role:** Sticky Note; visual annotation.
- **Configuration choices:** Explains that this is the core PageIndex reasoning step using the tree index rather than vector search.
- **Key expressions or variables used:** Refers to user question plus `doc_ids`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None beyond API behavior.
- **Sub-workflow reference:** None.

#### LLM Reasoning over Document Tree
- **Type and technical role:** `n8n-nodes-base.httpRequest`; POST request to PageIndex chat completions endpoint.
- **Configuration choices:**
  - URL: `https://api.pageindex.ai/chat/completions`
  - Method: `POST`
  - Body type: JSON
  - Headers:
    - `api_key: YOUR_PAGEINDEX_API_KEY`
    - `Content-Type: application/json`
  - JSON body contains:
    - `messages`: one user message from Telegram text
    - `stream: false`
    - `doc_id`: serialized array from `doc_ids`
    - `temperature: 0.5`
    - `enable_citations: false`
- **Key expressions or variables used:**
  - `{{ $('Receive User Question').item.json.message.text }}`
  - `{{ JSON.stringify($json.doc_ids) }}`
- **Input and output connections:**
  - Input from **Extract Document IDs**
  - Output to **Send Answer to User**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Hardcoded API key placeholder must be replaced.
  - If `doc_ids` is empty, the API may return an error or low-quality answer.
  - The field name is configured as `doc_id` even though the value is an array; this assumes the API accepts that exact schema.
  - If `Receive User Question` did not contain `message.text`, the expression fails.
  - Response structure must contain `choices[0].message.content` or the next node breaks.
  - Citations are disabled despite the notes describing cited answers.
- **Sub-workflow reference:** None.

#### Sticky Note - Send Answer
- **Type and technical role:** Sticky Note; visual annotation.
- **Configuration choices:** Documents that the generated answer is returned to the original Telegram sender.
- **Key expressions or variables used:** Refers to `choices[0].message.content` and `message.from.id`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** Notes per-user reply routing.
- **Sub-workflow reference:** None.

#### Send Answer to User
- **Type and technical role:** `n8n-nodes-base.telegram`; sends a text message through Telegram.
- **Configuration choices:**
  - Text:
    - `={{ $json.choices[0].message.content }}`
  - Chat ID:
    - `={{ $('Receive User Question').item.json.message.from.id }}`
  - Uses Telegram credentials named **Telegram account**
- **Key expressions or variables used:**
  - `{{$json.choices[0].message.content}}`
  - `{{ $('Receive User Question').item.json.message.from.id }}`
- **Input and output connections:**
  - Input from **LLM Reasoning over Document Tree**
  - No downstream output
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Fails if the PageIndex response does not contain `choices[0].message.content`.
  - Reply may fail if Telegram rejects the target chat ID or message content.
  - Long answers may exceed Telegram message size limits.
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Summary | stickyNote | Overall workflow documentation |  |  | ## 🧠 Workflow Summary<br>**PageIndex Vectorless RAG Bot**<br>This workflow builds a vectorless RAG system using [PageIndex](https://pageindex.ai) — no embeddings, no vector database. PageIndex builds a hierarchical tree index (like a Table of Contents) from your PDF, and the LLM reasons over that tree to find precise answers.<br>**📄 Flow 1 — PDF Upload (Run Once)** Send a PDF file to the Telegram bot. It downloads and indexes it on PageIndex cloud.<br>**💬 Flow 2 — Q&A (Runs Every Time)** Send any question to the Telegram bot. It fetches all your indexed docs, passes them to PageIndex LLM reasoning, and returns a cited answer.<br>**🔑 Required Credentials** - Telegram Bot Token (via @BotFather) - PageIndex API Key (via dash.pageindex.ai)<br>**💡 How PageIndex Works** Instead of vector similarity search, PageIndex builds a tree of section summaries. The LLM reads the tree, decides which sections are relevant, retrieves only those, and generates the answer — with page citations included. |
| Sticky Note - Flow 1 Header | stickyNote | Section header for PDF upload flow |  |  | ## 📄 Flow 1 -> PDF Knowledge Upload<br>Run this flow by sending a PDF file to the Telegram bot. The PDF is downloaded and indexed on PageIndex cloud. Run once per document. |
| Sticky Note - Receive PDF | stickyNote | Documentation for PDF trigger |  |  | ### 📎 Receive PDF Document<br>**Purpose:** Entry point for PDF upload flow. Listens for incoming Telegram messages that contain a file/document.<br>**Input:** Telegram message with a PDF attachment<br>**Output:** `message.document.file_id` used to download the file<br>**Note:** Only send PDF files here. Other file types will cause errors downstream. |
| Sticky Note - Download PDF | stickyNote | Documentation for Telegram file download |  |  | ### ⬇️ Download PDF File<br>**Purpose:** Downloads the PDF binary from Telegram's file storage using the `file_id`.<br>**Input:** `message.document.file_id` from Telegram Trigger<br>**Output:** Binary PDF file data<br>**Note:** Telegram stores files temporarily. This node fetches the actual file content before it expires. |
| Sticky Note - Index PDF | stickyNote | Documentation for PageIndex upload |  |  | ### ☁️ Index PDF on PageIndex<br>**Purpose:** Uploads the PDF to PageIndex cloud. PageIndex builds a hierarchical tree index (TOC) using LLM - no vectors.<br>**Input:** Binary PDF data<br>**Output:** `doc_id` (e.g. `pi-abc123`) assigned to this document<br>**Note:** Processing takes 10–60s depending on PDF size. The doc is stored permanently. |
| Receive PDF Document | telegramTrigger | Receives Telegram messages intended to contain PDFs |  | Download PDF File | ### 📎 Receive PDF Document<br>**Purpose:** Entry point for PDF upload flow. Listens for incoming Telegram messages that contain a file/document.<br>**Input:** Telegram message with a PDF attachment<br>**Output:** `message.document.file_id` used to download the file<br>**Note:** Only send PDF files here. Other file types will cause errors downstream. |
| Download PDF File | telegram | Downloads the Telegram file binary using file_id | Receive PDF Document | Index PDF on PageIndex | ### ⬇️ Download PDF File<br>**Purpose:** Downloads the PDF binary from Telegram's file storage using the `file_id`.<br>**Input:** `message.document.file_id` from Telegram Trigger<br>**Output:** Binary PDF file data<br>**Note:** Telegram stores files temporarily. This node fetches the actual file content before it expires. |
| Index PDF on PageIndex | httpRequest | Uploads the PDF to PageIndex for indexing | Download PDF File |  | ### ☁️ Index PDF on PageIndex<br>**Purpose:** Uploads the PDF to PageIndex cloud. PageIndex builds a hierarchical tree index (TOC) using LLM - no vectors.<br>**Input:** Binary PDF data<br>**Output:** `doc_id` (e.g. `pi-abc123`) assigned to this document<br>**Note:** Processing takes 10–60s depending on PDF size. The doc is stored permanently. |
| Sticky Note - Flow 2 Header | stickyNote | Section header for Q&A flow |  |  | ## 💬 Flow 2 -> Q&A Chat<br>Run this flow by sending any text message to the Telegram bot. It fetches all indexed docs, sends the question to PageIndex LLM reasoning, and returns a cited answer to the same user. |
| Sticky Note - Receive Question | stickyNote | Documentation for question trigger |  |  | ### 💬 Receive User Question<br>**Purpose:** Entry point for Q&A flow. Listens for text messages sent to the Telegram bot.<br>**Input:** Telegram text message from user<br>**Output:** `message.text` (the question) and `message.from.id` (for reply routing)<br>**Note:** Any text message triggers this flow. The answer is always sent back to the same user. |
| Sticky Note - Fetch Docs | stickyNote | Documentation for PageIndex document listing |  |  | ### 📚 Fetch All Indexed Documents<br>**Purpose:** Retrieves the list of all documents already indexed in your PageIndex account.<br>**Input:** PageIndex API key (header)<br>**Output:** Array of document objects including `id`, `name`, `status`, `pageNum`<br>**Note:** Only documents with `status: completed` are ready for querying. |
| Sticky Note - Extract IDs | stickyNote | Documentation for doc ID extraction |  |  | ### 🔑 Extract Document IDs<br>**Purpose:** Maps the documents array into a clean array of `doc_id` strings for use in the chat API.<br>**Input:** `documents` array from PageIndex<br>**Output:** `doc_ids` array (e.g. `["pi-abc123", "pi-def456"]`)<br>**Note:** Passing multiple doc_ids allows PageIndex to reason across all your indexed documents at once. |
| Sticky Note - LLM Reasoning | stickyNote | Documentation for PageIndex chat reasoning |  |  | ### 🧠 LLM Reasoning over Document Tree<br>**Purpose:** Core RAG step. Sends the user question + all doc_ids to PageIndex. The LLM reasons over the hierarchical tree index to find and retrieve the most relevant sections.<br>**Input:** User question + `doc_ids` array<br>**Output:** LLM-generated answer with page citations<br>**Note:** No vector search involved. PageIndex uses tree traversal + LLM reasoning. |
| Sticky Note - Send Answer | stickyNote | Documentation for Telegram reply |  |  | ### 📤 Send Answer to User<br>**Purpose:** Delivers the LLM-generated answer back to the user on Telegram.<br>**Input:** `choices[0].message.content` from PageIndex response<br>**Output:** Text message sent to the user's Telegram chat<br>**Note:** Reply is routed using `message.from.id` so each user gets their own response. |
| Receive User Question | telegramTrigger | Receives Telegram messages intended to be user questions |  | Fetch All Indexed Documents | ### 💬 Receive User Question<br>**Purpose:** Entry point for Q&A flow. Listens for text messages sent to the Telegram bot.<br>**Input:** Telegram text message from user<br>**Output:** `message.text` (the question) and `message.from.id` (for reply routing)<br>**Note:** Any text message triggers this flow. The answer is always sent back to the same user. |
| Fetch All Indexed Documents | httpRequest | Lists all indexed PageIndex documents | Receive User Question | Extract Document IDs | ### 📚 Fetch All Indexed Documents<br>**Purpose:** Retrieves the list of all documents already indexed in your PageIndex account.<br>**Input:** PageIndex API key (header)<br>**Output:** Array of document objects including `id`, `name`, `status`, `pageNum`<br>**Note:** Only documents with `status: completed` are ready for querying. |
| Extract Document IDs | set | Builds an array of document IDs from the PageIndex response | Fetch All Indexed Documents | LLM Reasoning over Document Tree | ### 🔑 Extract Document IDs<br>**Purpose:** Maps the documents array into a clean array of `doc_id` strings for use in the chat API.<br>**Input:** `documents` array from PageIndex<br>**Output:** `doc_ids` array (e.g. `["pi-abc123", "pi-def456"]`)<br>**Note:** Passing multiple doc_ids allows PageIndex to reason across all your indexed documents at once. |
| LLM Reasoning over Document Tree | httpRequest | Sends the question and document IDs to PageIndex chat completions | Extract Document IDs | Send Answer to User | ### 🧠 LLM Reasoning over Document Tree<br>**Purpose:** Core RAG step. Sends the user question + all doc_ids to PageIndex. The LLM reasons over the hierarchical tree index to find and retrieve the most relevant sections.<br>**Input:** User question + `doc_ids` array<br>**Output:** LLM-generated answer with page citations<br>**Note:** No vector search involved. PageIndex uses tree traversal + LLM reasoning. |
| Send Answer to User | telegram | Sends the generated answer back to the Telegram user | LLM Reasoning over Document Tree |  | ### 📤 Send Answer to User<br>**Purpose:** Delivers the LLM-generated answer back to the user on Telegram.<br>**Input:** `choices[0].message.content` from PageIndex response<br>**Output:** Text message sent to the user's Telegram chat<br>**Note:** Reply is routed using `message.from.id` so each user gets their own response. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it something like: `PageIndex RAG - Vectorless PDF Q&A`.

2. **Create Telegram credentials.**
   - In n8n, create a credential for Telegram.
   - Use the bot token obtained from **@BotFather**.
   - Save it as something like `Telegram account`.

3. **Prepare your PageIndex API key.**
   - Obtain it from `dash.pageindex.ai`.
   - In this workflow JSON, the API key is hardcoded in HTTP headers as `YOUR_PAGEINDEX_API_KEY`.
   - Replace that placeholder in every PageIndex HTTP node.
   - A more maintainable alternative is to store it in an n8n credential or environment variable.

4. **Add the first trigger node: `Receive PDF Document`.**
   - Node type: **Telegram Trigger**
   - Set it to listen for `message` updates.
   - Attach the `Telegram account` credential.
   - This node is the entry point for file uploads.

5. **Add the second node in Flow 1: `Download PDF File`.**
   - Node type: **Telegram**
   - Resource: **file**
   - File ID: `={{ $json.message.document.file_id }}`
   - Use the same `Telegram account` credential.
   - Connect:
     - `Receive PDF Document` → `Download PDF File`

6. **Add the third node in Flow 1: `Index PDF on PageIndex`.**
   - Node type: **HTTP Request**
   - Method: `POST`
   - URL: `https://api.pageindex.ai/doc/`
   - Enable sending headers.
   - Add header:
     - `api_key` = your real PageIndex API key
   - Enable sending body.
   - Content type: **multipart-form-data**
   - Add one form body field:
     - Name: `file`
     - Parameter type: **formBinaryData**
     - Input data field name: `data`
   - Connect:
     - `Download PDF File` → `Index PDF on PageIndex`

7. **Add the second trigger node: `Receive User Question`.**
   - Node type: **Telegram Trigger**
   - Set it to listen for `message` updates.
   - Use the same `Telegram account` credential.
   - This is the entry point for the Q&A path.

8. **Add `Fetch All Indexed Documents`.**
   - Node type: **HTTP Request**
   - Method: `GET` (default is acceptable if unchanged)
   - URL: `https://api.pageindex.ai/docs`
   - Enable sending headers.
   - Add header:
     - `api_key` = your real PageIndex API key
   - Connect:
     - `Receive User Question` → `Fetch All Indexed Documents`

9. **Add `Extract Document IDs`.**
   - Node type: **Set**
   - Create one field:
     - Name: `doc_ids`
     - Type: `Array`
     - Value: `={{ $json.documents.map(d => d.id) }}`
   - Connect:
     - `Fetch All Indexed Documents` → `Extract Document IDs`

10. **Add `LLM Reasoning over Document Tree`.**
    - Node type: **HTTP Request**
    - Method: `POST`
    - URL: `https://api.pageindex.ai/chat/completions`
    - Enable sending headers.
    - Add headers:
      - `api_key` = your real PageIndex API key
      - `Content-Type` = `application/json`
    - Enable sending body.
    - Set body type to **JSON**.
    - Use this logical structure for the JSON body:
      - `messages`: a single user message containing Telegram text from the question trigger
      - `stream`: `false`
      - `doc_id`: the serialized `doc_ids` array
      - `temperature`: `0.5`
      - `enable_citations`: `false`
    - Configure expressions exactly as follows:
      - Question text: `{{ $('Receive User Question').item.json.message.text }}`
      - Doc IDs array serialization: `{{ JSON.stringify($json.doc_ids) }}`
    - Connect:
      - `Extract Document IDs` → `LLM Reasoning over Document Tree`

11. **Add `Send Answer to User`.**
    - Node type: **Telegram**
    - Operation: send text message
    - Text: `={{ $json.choices[0].message.content }}`
    - Chat ID: `={{ $('Receive User Question').item.json.message.from.id }}`
    - Use the `Telegram account` credential.
    - Connect:
      - `LLM Reasoning over Document Tree` → `Send Answer to User`

12. **Optionally add sticky notes** to match the original workflow structure.
    - Add one summary note describing the system and required credentials.
    - Add one header note for Flow 1.
    - Add one header note for Flow 2.
    - Add node-level notes for upload, download, indexing, fetching docs, extracting IDs, reasoning, and replying.

13. **Check n8n workflow settings.**
    - Binary mode in the original workflow is set to `separate`.
    - Execution order is `v1`.
    - These are not always mandatory to reproduce behavior, but matching them may help consistency.

14. **Test Flow 1 with a PDF upload.**
    - Send a PDF file to the Telegram bot.
    - Confirm that:
      - the Telegram trigger fires,
      - the file is downloaded,
      - the HTTP upload to PageIndex succeeds.

15. **Wait for indexing to complete** before testing Q&A.
    - The sticky note indicates indexing may take **10–60 seconds**, depending on document size.
    - If you ask questions too soon, the document may not yet be queryable.

16. **Test Flow 2 with a text question.**
    - Send a plain text question to the Telegram bot.
    - Confirm that:
      - the workflow retrieves PageIndex docs,
      - extracts IDs,
      - calls the chat endpoint,
      - sends the answer back in Telegram.

17. **Recommended hardening if rebuilding for production.**
    - Add an **IF** node after `Receive PDF Document` to confirm `message.document` exists and optionally validate MIME type.
    - Add an **IF** node after `Receive User Question` to ensure `message.text` exists.
    - Filter documents by `status === 'completed'` before extracting IDs.
    - Add error handling for empty document lists.
    - Consider enabling citations if the PageIndex API supports them in your account/plan.
    - Replace hardcoded API key values with a secure credential pattern.

### Sub-workflow setup
- This workflow contains **no Execute Workflow node** and **no sub-workflow invocation**.
- There are **two independent entry points**:
  1. `Receive PDF Document`
  2. `Receive User Question`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PageIndex website referenced in the workflow summary | https://pageindex.ai |
| PageIndex dashboard is referenced as the source for the API key | `dash.pageindex.ai` |
| Telegram Bot Token is expected to be created via BotFather | Telegram / `@BotFather` |
| The workflow description claims cited answers, but the actual chat request sets `enable_citations` to `false` | Important implementation discrepancy |
| Both Telegram triggers listen to the same `message` update type without explicit filtering | Important design consideration |
| The workflow is inactive in the provided export (`active: false`) | Deployment state |
| No vector database or embeddings are used; retrieval is based on PageIndex hierarchical indexing and LLM reasoning | Architectural characteristic |