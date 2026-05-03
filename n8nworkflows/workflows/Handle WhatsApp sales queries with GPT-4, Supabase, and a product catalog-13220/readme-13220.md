Handle WhatsApp sales queries with GPT-4, Supabase, and a product catalog

https://n8nworkflows.xyz/workflows/handle-whatsapp-sales-queries-with-gpt-4--supabase--and-a-product-catalog-13220


# Handle WhatsApp sales queries with GPT-4, Supabase, and a product catalog

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow implements an AI-powered WhatsApp sales agent (“TURK”) for a furniture business. It receives WhatsApp messages (text, voice notes, images), converts non-text inputs to text (voice → transcription/translation; image → description), then uses a GPT-based agent with tools to:  
- search a product catalog stored in an n8n Data Table,  
- retrieve FAQs/policies from a Supabase vector knowledge base (RAG),  
- optionally perform web search via SerpAPI.  
Finally, it validates/cleans the AI output, detects whether an image URL is present, and sends either a text-only WhatsApp reply or an image-with-caption reply.

**Target use cases:**
- Customers ask about product availability/prices/specs via WhatsApp
- Customers send a photo of a furniture item and ask for similar products
- Customers send voice notes instead of typing
- Customers ask shipping/policy questions (answered from knowledge base)

### 1.1 Input Reception & Message-Type Routing
Receives inbound WhatsApp messages and routes them by type (text/audio/image/document).

### 1.2 Media Pre-processing (Audio + Image)
Downloads WhatsApp media, transcribes voice notes with OpenAI, and analyzes images with OpenAI Vision to produce a text prompt.

### 1.3 AI Sales Agent (LLM + Tools + Memory)
Runs a LangChain Agent backed by OpenAI chat model, equipped with tools:
- n8n Data Table product search tool
- Supabase vector store retrieval tool (RAG FAQs/policies)
- SerpAPI web search tool  
and conversation memory keyed by WhatsApp user.

### 1.4 Response Validation & WhatsApp Reply Formatting
Parses AI output to remove artifacts and extract an image URL (if any), then routes to the correct WhatsApp send method (text vs image+caption).

### 1.5 Knowledge Base Ingestion (Form Upload → Supabase Vector Store)
A separate entry point allows uploading FAQ docs via an n8n form; documents are embedded and inserted into Supabase vector store for later retrieval.

---

## 2. Block-by-Block Analysis

### Block 2.1 — WhatsApp Entry & Type Routing
**Overview:** Receives inbound WhatsApp messages and branches processing based on message type (text, audio, image, document).  
**Nodes involved:** WhatsApp Trigger, Route Types, Map text prompt, Send message1

#### Node: WhatsApp Trigger
- **Type / role:** `whatsAppTrigger` — webhook trigger for WhatsApp Business events.
- **Configuration (interpreted):**
  - Watches **updates:** `messages`
  - Retries enabled (`retryOnFail: true`)
- **Key data used later:**
  - `messages[0].type`, `messages[0].from`
  - `metadata.phone_number_id`
  - `contacts[0].wa_id`, `contacts[0].profile.name`
- **Outputs:** to **Route Types**
- **Failure/edge cases:**
  - Invalid/expired WhatsApp OAuth credentials
  - Payload shape differences (missing `contacts[0]` or `messages[0]`)
  - Multiple messages in one webhook (workflow assumes index `[0]`)

#### Node: Route Types
- **Type / role:** `switch` — routes by `messages[0].type`
- **Configuration:**
  - Rules (renamed outputs):
    - **Text** if `{{$json.messages[0].type}} == "text"`
    - **Audio** if `... == "audio"`
    - **Image** if `... == "image"`
    - **Document** if `... == "document"`
  - Fallback output: `none` (drops unmatched types)
- **Outputs:**
  - Text → **Map text prompt**
  - Audio → **Gets WhatsApp Voicemail Source URL**
  - Image → **Gets WhatsApp Image Source URL**
  - Document → **Send message1**
- **Failure/edge cases:**
  - If `messages[0].type` is missing, switch may route to none (no response)
  - WhatsApp message types like `button`, `interactive`, `location` are not handled

#### Node: Map text prompt
- **Type / role:** `set` — normalizes inbound text into a `text` field for the agent
- **Configuration:**
  - Sets `text = {{$json.messages[0].text.body}}`
- **Outputs:** → **Sales AI agent**
- **Failure/edge cases:**
  - If `messages[0].text.body` missing (some WhatsApp payload variants), expression fails or results in empty prompt

#### Node: Send message1
- **Type / role:** `whatsApp` send — fallback for unsupported document uploads
- **Configuration:**
  - Sends text: `"Please we do not support document upload"`
  - `phoneNumberId = {{ $('WhatsApp Trigger').item.json.metadata.phone_number_id }}`
  - `recipientPhoneNumber = {{ $('WhatsApp Trigger').item.json.messages[0].from }}`
- **Inputs:** from Route Types (Document output)
- **Failure/edge cases:**
  - WhatsApp send permission/phone number ID mismatch
  - If trigger data missing, expressions break

**Sticky note applied (context):**
- “## fall back response” (covers this general fallback area)

---

### Block 2.2 — Audio Handling (Voice Note → Text)
**Overview:** For audio messages, fetches the media URL from WhatsApp, downloads the audio, and uses OpenAI audio translation/transcription to produce text for the agent.  
**Nodes involved:** Gets WhatsApp Voicemail Source URL, Download Voicemail, OpenAI, Sales AI agent

#### Node: Gets WhatsApp Voicemail Source URL
- **Type / role:** `whatsApp` (mediaUrlGet) — retrieves a temporary download URL for the audio media ID.
- **Configuration:**
  - `mediaGetId = {{ $json.messages[0].audio.id }}`
- **Outputs:** → **Download Voicemail**
- **Failure/edge cases:**
  - Media ID expired or invalid
  - WhatsApp API permissions
  - Expression typo risk: note the expression uses `audio.id}}` with no space; functionally ok but depends on payload presence

#### Node: Download Voicemail
- **Type / role:** `httpRequest` — downloads the media from the retrieved URL
- **Configuration:**
  - `url = {{ $json.url }}`
  - Auth: predefined credential type `whatsAppApi`
- **Outputs:** → **OpenAI** (audio)
- **Failure/edge cases:**
  - URL expiration (WhatsApp media URLs can be short-lived)
  - Large audio file timeouts
  - Incorrect credential selection (must authorize WhatsApp media download)

#### Node: OpenAI (audio translate)
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — audio translation operation (speech-to-text).
- **Configuration:**
  - `resource: audio`
  - `operation: translate`
- **Outputs:** → **Sales AI agent**
- **Failure/edge cases:**
  - Unsupported audio codecs/containers
  - Size limits / rate limits
  - Credential/auth issues with OpenAI
  - Output field naming: downstream expects agent input `text` but audio node output commonly uses fields like `text`/`output` depending on node version—here it is directly connected to agent; ensure agent is reading from `$json` appropriately.

**Sticky note applied (context):**
- “## Upload Audio files / For processing audio”

---

### Block 2.3 — Image Handling (Image → Description Text)
**Overview:** For image messages, fetches the media URL, downloads the image, sends it to OpenAI Vision for analysis, then maps the description + WhatsApp caption into a single text prompt.  
**Nodes involved:** Gets WhatsApp Image Source URL, Download Image, OpenAI1, Map image prompt, Sales AI agent

#### Node: Gets WhatsApp Image Source URL
- **Type / role:** `whatsApp` (mediaUrlGet) — retrieves temporary URL for image media ID.
- **Configuration:**
  - `mediaGetId = {{ $json.messages[0].image.id }}`
- **Outputs:** → **Download Image**
- **Failure/edge cases:**
  - Media ID expired/invalid
  - Customer sends multiple images (workflow only uses `messages[0]`)

#### Node: Download Image
- **Type / role:** `httpRequest` — downloads image binary from URL
- **Configuration:**
  - `url = {{ $json.url }}`
  - Auth: predefined `whatsAppApi`
- **Outputs:** → **OpenAI1**
- **Failure/edge cases:**
  - URL expiration
  - Unsupported formats
  - Large images causing timeouts

#### Node: OpenAI1 (image analyze)
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — vision analysis to text
- **Configuration:**
  - `resource: image`, `operation: analyze`
  - `model: gpt-4o-mini`
  - Prompt text: “What’s in this image? describe in relation to furnitures”
  - `inputType: base64`, `detail: auto`
- **Outputs:** → **Map image prompt**
- **Failure/edge cases:**
  - Image not properly passed as base64 (depends on HTTP Request output/binary mapping)
  - Model availability/permissions
  - Vision response field naming differences across versions

#### Node: Map image prompt
- **Type / role:** `set` — creates unified `text` prompt from image description + caption
- **Configuration:**
  - `text = "User image description: {{ $json.content }}\n\nUser image caption: {{ $('Route Types').item.json.messages[0].image.caption }}"`
- **Outputs:** → **Sales AI agent**
- **Failure/edge cases:**
  - If OpenAI vision output doesn’t contain `$json.content` (could be `$json.output`/`$json.text` depending on version)
  - If caption is absent, expression may return `undefined`

**Sticky note applied (context):**
- “## Upload image files / For processing images”

---

### Block 2.4 — AI Agent Core (LLM + Tools + Memory)
**Overview:** Uses a chat model with memory and multiple tools (product catalog search, FAQ retrieval, web search) to generate a sales response aligned with a long system prompt.  
**Nodes involved:** Sales AI agent, the brain, System Memory, Knowledgebase, Web Search, search_products_inventory

#### Node: the brain
- **Type / role:** `lmChatOpenAi` — provides the chat language model to the agent
- **Configuration:**
  - Model: `gpt-4.1-mini`
- **Outputs (special connection):**
  - `ai_languageModel` → **Sales AI agent**
- **Failure/edge cases:**
  - OpenAI rate limits
  - Model not enabled on the account

#### Node: System Memory
- **Type / role:** `memoryBufferWindow` — short-term conversation memory per WhatsApp user
- **Configuration:**
  - `sessionIdType: customKey`
  - `sessionKey = memory_{{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
  - `contextWindowLength: 20` (keeps last ~20 turns/items depending on node behavior)
- **Outputs (special connection):**
  - `ai_memory` → **Sales AI agent**
- **Failure/edge cases:**
  - Missing `contacts[0].wa_id` breaks session key expression
  - Memory growth/behavior depends on node version; windowed memory reduces risk but still might store sensitive info—consider retention policies

#### Node: Knowledgebase
- **Type / role:** `vectorStoreSupabase` (retrieve-as-tool) — RAG retrieval tool for FAQs/policies
- **Configuration:**
  - Mode: “retrieve as tool”
  - Table: `documents`
  - Query function name: `match_documents`
  - Tool description: “Use this tool to retrieve the knowledge about customCX”
- **Dependencies:**
  - Requires embeddings connection from **Embeddings OpenAI1**
- **Outputs (special connection):**
  - `ai_tool` → **Sales AI agent**
- **Failure/edge cases:**
  - Supabase credentials invalid
  - Table/function `documents` / `match_documents` not created or mismatched schema
  - Embedding dimension mismatch between stored vectors and runtime embeddings

#### Node: Web Search
- **Type / role:** `toolSerpApi` — web search tool
- **Configuration:**
  - `google_domain: google.com`
- **Outputs (special connection):**
  - `ai_tool` → **Sales AI agent**
- **Failure/edge cases:**
  - SerpAPI quota/limits
  - Regional restrictions, inconsistent results

#### Node: search_products_inventory
- **Type / role:** `dataTableTool` — tool wrapper over n8n Data Tables (product catalog)
- **Configuration:**
  - Operation: `get`
  - `returnAll: true`
  - Data Table: “ISTANBUL TURKISH FURNITURE”
- **Outputs (special connection):**
  - `ai_tool` → **Sales AI agent**
- **Failure/edge cases:**
  - Data Table not present in the target n8n instance
  - Returning all rows can be heavy; agent prompt/tool call may become slow or exceed token constraints
  - “get” operation semantics: if it fetches all records without filtering, the agent may struggle; ideally requires query/filter parameters (depends on node implementation)

#### Node: Sales AI agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates tool use + LLM response generation
- **Configuration:**
  - Input text: `whatsapp message:{{ $json.text }}`
  - System message: very long “TURK - AI Sales Agent System Prompt v2.0” including:
    - strict rule to use `search_products_inventory` for product info
    - guidance for shipping answers via knowledge base
    - instruction: “ALWAYS execute removeAnnotations_tool”
- **Connections:**
  - Receives `main` input from:
    - **Map text prompt**
    - **OpenAI** (audio)
    - **Map image prompt**
  - Receives `ai_languageModel` from **the brain**
  - Receives `ai_memory` from **System Memory**
  - Receives `ai_tool` from **Knowledgebase**, **Web Search**, **search_products_inventory**
  - Outputs `main` → **Response_validation**
- **Important integration note:**  
  The system prompt references **`removeAnnotations_tool`**, but **no such tool node exists** in the workflow. The only cleaning step present is the **Response_validation** code node *after* the agent. This mismatch can confuse the agent (it may attempt a non-existent tool call).
- **Failure/edge cases:**
  - Agent tool-call failures (timeouts, tool errors)
  - Token/context overflow due to long system prompt + memory + large tool outputs
  - Hallucinated prices if product tool returns too much or unclear results (despite prompt rules)

**Sticky note applied (context):**
- “### How it works … Setup steps … Customization tips …” (overall workflow guidance)

---

### Block 2.5 — Response Cleaning, URL Extraction, and WhatsApp Sending
**Overview:** Cleans the agent output, extracts image URL if present, validates formatting, then routes to either text-only send or image-with-caption send.  
**Nodes involved:** Response_validation, If, Send text message, Send media&caption message

#### Node: Response_validation
- **Type / role:** `code` — parses and cleans AI response
- **Configuration (logic summary):**
  - Reads AI text from the first non-empty of: `output | message | response | text`
  - Removes:
    - ANSI escape codes (`\[\d+m`, `\x1b\[\d+m`)
    - bracketed annotations like `[thought]`, `[reasoning]`, `[internal]` (pattern: `\[[a-zA-Z_]+\]`)
    - excessive newlines
  - Extracts image URL using patterns in order:
    1. Custom `||IMG:url||`
    2. Markdown `![alt](url)`
    3. Specific domain `https://www.buyfromturkeynj.com/image/...(.jpg|png|...)`
    4. Generic image URL ending in common extensions
  - Outputs:
    - `message` (cleaned text)
    - `imageUrl` (string or null)
    - `hasImage` (boolean)
- **Outputs:** → **If**
- **Failure/edge cases:**
  - If AI output contains a non-image URL, the generic pattern won’t match (only image extensions)
  - If WhatsApp requires HTTPS and URL is HTTP, send may fail (code does not enforce HTTPS)
  - “Validation” is mostly parsing; it does not actually HEAD-check URL availability despite sticky note claim (“validates the urls”)

#### Node: If
- **Type / role:** `if` — decides whether to send text-only or media message
- **Configuration:**
  - Condition checks **empty**: `{{ $json.imageUrl }}` is empty
  - True branch (empty) → **Send text message**
  - False branch (has URL) → **Send media&caption message**
- **Failure/edge cases:**
  - `imageUrl: null` is treated as empty by the “empty” operator (desired)
  - If `imageUrl` is present but WhatsApp rejects it, media send fails; no fallback path implemented

#### Node: Send text message
- **Type / role:** `whatsApp` send — sends plain text
- **Configuration:**
  - `textBody = {{ $json.message }}`
  - `phoneNumberId = {{ $('WhatsApp Trigger').item.json.metadata.phone_number_id }}`
  - `recipientPhoneNumber = {{ $('WhatsApp Trigger').item.json.messages[0].from }}`
- **Failure/edge cases:**
  - WhatsApp length limits; long AI responses may be rejected or truncated
  - Missing trigger fields in expression

#### Node: Send media&caption message
- **Type / role:** `whatsApp` send — sends image with caption
- **Configuration:**
  - `messageType: image`
  - `mediaLink = {{ $json.imageUrl }}`
  - Caption: `mediaCaption = {{ $json.message }}`
  - Recipient/phoneNumberId from trigger
- **Failure/edge cases:**
  - WhatsApp requires the media URL to be publicly accessible and meet format/size requirements
  - Caption length limits
  - Broken/expired external URLs

**Sticky note applied (context):**
- “### Logic … Code node identifies if output has image urls … If node routes …”

---

### Block 2.6 — Knowledge Base (RAG) Ingestion via Form Upload
**Overview:** Provides a form to upload FAQ documents, then loads and splits them, generates embeddings, and inserts them into Supabase vector store.  
**Nodes involved:** On form submission1, Supabase Vector Store, Default Data Loader, Recursive Character Text Splitter, Embeddings OpenAI1

#### Node: On form submission1
- **Type / role:** `formTrigger` — manual KB ingestion entry point
- **Configuration:**
  - Form title: “Turkey Furnitures KB”
  - Field: `Knowledgebase` (file), required, accepts `.pdf`
  - Authentication: Basic Auth (credential “Naim”)
  - Description: “Upload FAQs docs”
- **Outputs:** → **Supabase Vector Store** (main)
- **Failure/edge cases:**
  - Accepts PDF, but downstream loader is configured as **docxLoader** (mismatch; see below)
  - Basic auth credential must be configured and shared securely

#### Node: Supabase Vector Store (insert)
- **Type / role:** `vectorStoreSupabase` (insert mode) — stores documents + embeddings into Supabase
- **Configuration:**
  - Mode: `insert`
  - Table: `documents`
  - Query name: `match_documents` (used for retrieval queries later)
- **Connections:**
  - Receives `main` from **On form submission1**
  - Receives `ai_document` from **Default Data Loader**
  - Receives `ai_embedding` from **Embeddings OpenAI1**
- **Failure/edge cases:**
  - Supabase table/schema not aligned with n8n vector store expectations
  - Embedding dimension mismatch (must match retrieval embeddings)
  - Insert errors if metadata fields conflict with schema

#### Node: Default Data Loader
- **Type / role:** `documentDefaultDataLoader` — converts uploaded file into text Documents
- **Configuration:**
  - Loader: **docxLoader**
  - Data type: `binary`
  - Metadata adds: `file_id = {{ $('Set the fields for mapping').item.json.file_id }}`
  - Text splitting mode: `custom` (uses an external splitter node)
- **Connections:**
  - Receives `ai_textSplitter` from **Recursive Character Text Splitter**
  - Outputs `ai_document` → **Supabase Vector Store**
- **Critical issue / edge case:**
  - The form accepts `.pdf`, but loader is `docxLoader`. For PDFs you typically need `pdfLoader`. As configured, PDF uploads may fail or produce empty text.
  - Metadata expression references a node **“Set the fields for mapping”** which does **not exist** in this workflow. This will fail at runtime unless the node exists in the actual environment or was renamed/removed.
- **Failure types:**
  - Expression evaluation error for missing node
  - Loader parsing error due to file type mismatch

#### Node: Recursive Character Text Splitter
- **Type / role:** recursive text splitter — chunks documents for embeddings
- **Configuration:** default options (not customized)
- **Outputs:** `ai_textSplitter` → **Default Data Loader**
- **Failure/edge cases:**
  - If chunk sizes/defaults are not tuned, could produce too-large chunks or too many chunks

#### Node: Embeddings OpenAI1
- **Type / role:** embeddings model — generates embeddings for RAG
- **Configuration:**
  - Dimensions: **1536** (explicit)
- **Connections:**
  - `ai_embedding` → **Knowledgebase** (retrieve tool)
  - `ai_embedding` → **Supabase Vector Store** (insert)
- **Version-specific requirement:**
  - Embedding dimensions must match:
    - the model used, and
    - the Supabase vector column dimension.
- **Failure/edge cases:**
  - If Supabase vector column dimension differs, inserts/queries fail
  - OpenAI embedding API limits/rate limits

**Sticky note applied (context):**
- “## RAG pipeline / Set the embedding Dimension to 1536 for openAI or 1024 for cohere”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| WhatsApp Trigger | whatsAppTrigger | Entry point: receive WhatsApp messages webhook | — | Route Types | ### How it works… Setup steps… Customization tips… |
| Route Types | switch | Route by message type (text/audio/image/document) | WhatsApp Trigger | Map text prompt; Gets WhatsApp Voicemail Source URL; Gets WhatsApp Image Source URL; Send message1 | ### How it works… Setup steps… Customization tips… |
| Map text prompt | set | Normalize text payload to `text` | Route Types (Text) | Sales AI agent | ### How it works… Setup steps… Customization tips… |
| Gets WhatsApp Voicemail Source URL | whatsApp | Fetch audio media download URL | Route Types (Audio) | Download Voicemail | ## Upload Audio files / For processing audio |
| Download Voicemail | httpRequest | Download voice note binary | Gets WhatsApp Voicemail Source URL | OpenAI | ## Upload Audio files / For processing audio |
| OpenAI | langchain.openAi | Speech-to-text translation/transcription | Download Voicemail | Sales AI agent | ## Upload Audio files / For processing audio |
| Gets WhatsApp Image Source URL | whatsApp | Fetch image media download URL | Route Types (Image) | Download Image | ## Upload image files / For processing images |
| Download Image | httpRequest | Download image binary | Gets WhatsApp Image Source URL | OpenAI1 | ## Upload image files / For processing images |
| OpenAI1 | langchain.openAi | Vision analysis (image→description) | Download Image | Map image prompt | ## Upload image files / For processing images |
| Map image prompt | set | Build `text` from vision description + caption | OpenAI1 | Sales AI agent | ## Upload image files / For processing images |
| Send message1 | whatsApp | Document upload not supported response | Route Types (Document) | — | ## fall back response |
| Sales AI agent | langchain.agent | Main agent: uses LLM + tools + memory to craft response | Map text prompt / OpenAI / Map image prompt | Response_validation | ### How it works… Setup steps… Customization tips… |
| the brain | lmChatOpenAi | Chat model provider for agent | — | Sales AI agent | ### How it works… Setup steps… Customization tips… |
| System Memory | memoryBufferWindow | Per-user conversation memory | — | Sales AI agent | ### How it works… Setup steps… Customization tips… |
| Knowledgebase | vectorStoreSupabase | RAG retrieval tool from Supabase | Embeddings OpenAI1 (embedding) | Sales AI agent | ## RAG pipeline / Set the embedding Dimension… |
| Web Search | toolSerpApi | Web search tool | — | Sales AI agent | ### How it works… Setup steps… Customization tips… |
| search_products_inventory | dataTableTool | Product catalog search tool (n8n Data Table) | — | Sales AI agent | ### How it works… Setup steps… Customization tips… |
| Response_validation | code | Clean AI output + extract image URL | Sales AI agent | If | ### Logic… Code node… If node routes… |
| If | if | Route to text vs image message | Response_validation | Send text message; Send media&caption message | ### Logic… Code node… If node routes… |
| Send text message | whatsApp | Send WhatsApp text response | If (true) | — | ### Logic… Code node… If node routes… |
| Send media&caption message | whatsApp | Send WhatsApp image + caption response | If (false) | — | ### Logic… Code node… If node routes… |
| On form submission1 | formTrigger | Entry point: upload KB docs | — | Supabase Vector Store | ## RAG pipeline / Set the embedding Dimension… |
| Supabase Vector Store | vectorStoreSupabase | Insert documents into Supabase vector DB | On form submission1; Default Data Loader; Embeddings OpenAI1 | — | ## RAG pipeline / Set the embedding Dimension… |
| Default Data Loader | documentDefaultDataLoader | Parse uploaded file to documents | Recursive Character Text Splitter | Supabase Vector Store | ## RAG pipeline / Set the embedding Dimension… |
| Recursive Character Text Splitter | textSplitterRecursiveCharacterTextSplitter | Chunk documents for embedding | — | Default Data Loader | ## RAG pipeline / Set the embedding Dimension… |
| Embeddings OpenAI1 | embeddingsOpenAi | Create embeddings (1536 dim) | — | Knowledgebase; Supabase Vector Store | ## RAG pipeline / Set the embedding Dimension… |
| Sticky Note6 | stickyNote | Comment | — | — | ### How it works… Setup steps… Customization tips… |
| Sticky Note3 | stickyNote | Comment | — | — | ## RAG pipeline / Set the embedding Dimension… |
| Sticky Note2 | stickyNote | Comment | — | — | ## Upload Audio files / For processing audio |
| Sticky Note | stickyNote | Comment | — | — | ## Upload image files / For processing images |
| Sticky Note1 | stickyNote | Comment | — | — | ## fall back response |
| Sticky Note4 | stickyNote | Comment | — | — | ### Logic… Code node… If node routes… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. WhatsApp Trigger OAuth credential (for `WhatsApp Trigger`).
   2. WhatsApp Business API credential (for `WhatsApp` send + `mediaUrlGet` + media download).
   3. OpenAI API credential (for chat model, embeddings, audio, vision).
   4. Supabase API credential (URL + service role key or appropriate key).
   5. SerpAPI credential.
   6. Basic Auth credential (for the KB upload form).

2) **Create the WhatsApp inbound flow**
   1. Add **WhatsApp Trigger** node:
      - Updates: `messages`
   2. Add **Switch** node named **Route Types**:
      - Add rules checking `{{$json.messages[0].type}}` equals `text`, `audio`, `image`, `document`
      - Enable renamed outputs (Text/Audio/Image/Document)
      - Fallback: none
   3. Connect: **WhatsApp Trigger → Route Types**

3) **Text path**
   1. Add **Set** node “Map text prompt”:
      - Field `text` = `{{$json.messages[0].text.body}}`
   2. Connect: **Route Types (Text) → Map text prompt**

4) **Audio path**
   1. Add **WhatsApp** node “Gets WhatsApp Voicemail Source URL”:
      - Resource: media
      - Operation: get media URL
      - Media ID: `{{$json.messages[0].audio.id}}`
   2. Add **HTTP Request** node “Download Voicemail”:
      - URL: `{{$json.url}}`
      - Authentication: predefined WhatsApp credential (must allow media download)
   3. Add **OpenAI** node (LangChain OpenAI) “OpenAI”:
      - Resource: audio
      - Operation: translate (speech-to-text)
   4. Connect: **Route Types (Audio) → Gets WhatsApp Voicemail Source URL → Download Voicemail → OpenAI**

5) **Image path**
   1. Add **WhatsApp** node “Gets WhatsApp Image Source URL”:
      - Resource: media
      - Operation: get media URL
      - Media ID: `{{$json.messages[0].image.id}}`
   2. Add **HTTP Request** node “Download Image”:
      - URL: `{{$json.url}}`
      - Authentication: predefined WhatsApp credential
      - Ensure response provides binary usable by the OpenAI image node (n8n binary handling)
   3. Add **OpenAI (Vision)** node “OpenAI1”:
      - Resource: image
      - Operation: analyze
      - Model: `gpt-4o-mini`
      - Input type: base64
      - Prompt: “What’s in this image? describe in relation to furnitures”
   4. Add **Set** node “Map image prompt”:
      - Field `text` =  
        `User image description: {{ $json.content }}\n\nUser image caption: {{ $('Route Types').item.json.messages[0].image.caption }}`
   5. Connect: **Route Types (Image) → Gets WhatsApp Image Source URL → Download Image → OpenAI1 → Map image prompt**

6) **Document fallback**
   1. Add **WhatsApp** node “Send message1”:
      - Operation: send
      - Text: “Please we do not support document upload”
      - phoneNumberId: `{{ $('WhatsApp Trigger').item.json.metadata.phone_number_id }}`
      - recipientPhoneNumber: `{{ $('WhatsApp Trigger').item.json.messages[0].from }}`
   2. Connect: **Route Types (Document) → Send message1**

7) **Create the agent toolchain**
   1. Add **LM Chat OpenAI** node “the brain”:
      - Model: `gpt-4.1-mini`
   2. Add **Memory Buffer Window** node “System Memory”:
      - Session ID type: custom key
      - Session key: `memory_{{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
      - Context window length: `20`
   3. Add **Data Table Tool** node “search_products_inventory”:
      - Select your Data Table containing catalog (SKU/name/model/price/links/images)
      - Operation: get
      - Return all: true (as per workflow; consider adding filtering later)
   4. Add **SerpAPI Tool** node “Web Search”:
      - google_domain: `google.com`
   5. Add **Embeddings OpenAI** node “Embeddings OpenAI1”:
      - Dimensions: 1536
   6. Add **Supabase Vector Store** node “Knowledgebase”:
      - Mode: retrieve-as-tool
      - Table: `documents`
      - Query name: `match_documents`
      - Set tool description (optional but recommended)
   7. Connect embeddings:
      - **Embeddings OpenAI1 (ai_embedding) → Knowledgebase (ai_embedding)**

8) **Create Sales AI agent**
   1. Add **LangChain Agent** node “Sales AI agent”:
      - Text input: `whatsapp message:{{ $json.text }}`
      - System message: paste your TURK system prompt (as in workflow)
   2. Connect:
      - **Map text prompt → Sales AI agent (main)**
      - **OpenAI (audio) → Sales AI agent (main)**
      - **Map image prompt → Sales AI agent (main)**
      - **the brain → Sales AI agent (ai_languageModel)**
      - **System Memory → Sales AI agent (ai_memory)**
      - **Knowledgebase → Sales AI agent (ai_tool)**
      - **Web Search → Sales AI agent (ai_tool)**
      - **search_products_inventory → Sales AI agent (ai_tool)**

   **Important:** Either remove “removeAnnotations_tool” from the system prompt or actually implement a tool with that name. In this workflow, cleaning is done later by the `Response_validation` node.

9) **Add response parsing and WhatsApp sending**
   1. Add **Code** node “Response_validation”:
      - Paste the parsing/cleaning code (logic: choose response field, strip annotations/ANSI, extract image URL, output `message`, `imageUrl`, `hasImage`)
   2. Add **If** node:
      - Condition: `{{$json.imageUrl}}` is empty
      - True → text send; False → media send
   3. Add **WhatsApp** node “Send text message”:
      - Operation: send
      - Text: `{{$json.message}}`
      - phoneNumberId: `{{ $('WhatsApp Trigger').item.json.metadata.phone_number_id }}`
      - recipientPhoneNumber: `{{ $('WhatsApp Trigger').item.json.messages[0].from }}`
   4. Add **WhatsApp** node “Send media&caption message”:
      - Operation: send
      - Message type: image
      - Media link: `{{$json.imageUrl}}`
      - Caption: `{{$json.message}}`
      - phoneNumberId / recipient as above
   5. Connect:
      - **Sales AI agent → Response_validation → If**
      - **If (true) → Send text message**
      - **If (false) → Send media&caption message**

10) **