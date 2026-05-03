Create an AI Telegram bot using Google Drive, Qdrant, and OpenAI GPT-4.1

https://n8nworkflows.xyz/workflows/create-an-ai-telegram-bot-using-google-drive--qdrant--and-openai-gpt-4-1-12228


# Create an AI Telegram bot using Google Drive, Qdrant, and OpenAI GPT-4.1

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow builds an AI-powered Telegram bot that answers user questions using a **Qdrant vector knowledge base** populated from documents automatically ingested from **Google Drive**, and generates responses using a **fine-tuned OpenAI GPT‑4.1 chat model**. It contains **two independent entry points** (two flows) that share the same embeddings + Qdrant collection.

**Target use cases:**
- Turn a folder of docs (PDF/DOCX/TXT, etc.) into a searchable knowledge base
- Provide a Telegram Q&A bot restricted to specific authorized users
- Maintain consistent style and leverage past materials via vector retrieval

### 1.1 Document Processing Flow (Google Drive → Qdrant)
- Watches an “incoming” Google Drive folder
- Downloads new files
- Splits text into chunks
- Creates embeddings and **inserts vectors into Qdrant**
- Moves processed files into a “Ready/Processed” folder

### 1.2 Telegram Chat Flow (Telegram → Agent → Qdrant tool → Telegram)
- Receives Telegram messages
- Filters to allow only an authorized chat/user ID
- Runs a LangChain-style **AI Agent** that can call Qdrant retrieval as a tool
- Sends the generated answer back to Telegram

### 1.3 Shared AI Components
- OpenAI Embeddings node used by both Qdrant insert and Qdrant retrieval tool
- Qdrant “retrieve-as-tool” node attached to the agent
- OpenAI Chat Model node providing the LLM for the agent (fine-tuned GPT‑4.1)

---

## 2. Block-by-Block Analysis

### Block A — Document Processing Flow (Google Drive ingestion → chunking → Qdrant insert → move file)

**Overview:**  
This block automatically ingests newly created files from a specific Google Drive folder, converts them to text chunks, embeds them with OpenAI embeddings, stores them in a Qdrant collection, and finally moves the file to a processed folder.

**Nodes involved:**
- New File Trigger
- Download File
- Insert into Qdrant
- Split Text into Chunks
- Load Document Data
- OpenAI Embeddings
- Move to Processed Folder

#### A1) New File Trigger
- **Type / role:** Google Drive Trigger (`n8n-nodes-base.googleDriveTrigger`) — entry point for document ingestion.
- **Configuration (interpreted):**
  - Event: **fileCreated**
  - Watches: **Specific folder** (ID `1FHG5g1T1CyctyPwwVsc64E-uIOlMLcTl`, labeled “Qdranttest”)
  - Polling interval: **every 15 minutes**
- **Key expressions/variables:** None; outputs file metadata including `id`.
- **Connections:**
  - Output → **Download File**
- **Version notes:** TypeVersion `1` (older trigger generation; polling-based).
- **Edge cases / failures:**
  - OAuth token expiry / missing Drive permissions
  - Polling delay (up to 15 min) can miss “instant” expectations
  - If a file is created but not yet fully uploaded, download may fail or be partial (rare but possible)

#### A2) Download File
- **Type / role:** Google Drive (`n8n-nodes-base.googleDrive`) — downloads file binary.
- **Configuration:**
  - Operation: **download**
  - File ID: `={{ $json.id }}` (from trigger output)
- **Connections:**
  - Input: **New File Trigger**
  - Output → **Insert into Qdrant**
- **Version notes:** TypeVersion `3`
- **Edge cases / failures:**
  - Download of unsupported Google-native formats (Docs/Sheets) may require “export” rather than “download”
  - Large files can cause memory/time issues depending on n8n instance limits

#### A3) Split Text into Chunks
- **Type / role:** Recursive Character Text Splitter (`@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`) — chunking strategy for long text.
- **Configuration:**
  - `chunkSize`: **3000**
  - `chunkOverlap`: **300**
- **Connections (special LangChain port):**
  - Output (`ai_textSplitter`) → **Load Document Data**
- **Important wiring note:** This node is not connected via “main”; it is connected via the **AI text splitter channel** used by LangChain document loader/processing chain.
- **Edge cases / failures:**
  - If text extraction yields empty text, splitter produces 0 chunks → Qdrant insert may insert nothing

#### A4) Load Document Data
- **Type / role:** Document Loader (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`) — converts binary into text/documents for downstream vectorization.
- **Configuration:**
  - Data type: **binary**
- **Connections:**
  - Receives splitter via `ai_textSplitter`
  - Output (`ai_document`) → **Insert into Qdrant**
- **Edge cases / failures:**
  - Unsupported file types or corrupted binary
  - Missing binary property on item (if upstream did not provide binary as expected)

#### A5) OpenAI Embeddings
- **Type / role:** OpenAI embeddings provider (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`) — shared embeddings model for Qdrant.
- **Configuration:**
  - Uses OpenAI API credentials
  - No explicit model specified (will use node default configured by the node/version or credential defaults)
- **Connections:**
  - Output (`ai_embedding`) → **Insert into Qdrant**
  - Output (`ai_embedding`) → **Qdrant Knowledge Base** (so retrieval uses the same embedding configuration)
- **Version notes:** TypeVersion `1.2`
- **Edge cases / failures:**
  - OpenAI auth errors, quota limits, rate limits
  - Large batch embedding may exceed token/size constraints depending on actual model defaults

#### A6) Insert into Qdrant
- **Type / role:** Qdrant Vector Store (`@n8n/n8n-nodes-langchain.vectorStoreQdrant`) — inserts chunks as vectors into a Qdrant collection.
- **Configuration:**
  - Mode: **insert**
  - Collection: **`Test-youtube-adept-ecom`**
  - Embedding batch size: **100**
  - Requires both:
    - Documents (`ai_document`) from **Load Document Data**
    - Embeddings (`ai_embedding`) from **OpenAI Embeddings**
- **Connections:**
  - Main input: **Download File** (main flow item context)
  - AI document input: **Load Document Data**
  - AI embedding input: **OpenAI Embeddings**
  - Main output → **Move to Processed Folder**
- **Version notes:** TypeVersion `1.3`
- **Edge cases / failures:**
  - Qdrant API key/URL misconfiguration, collection missing
  - Vector dimensionality mismatch (if embeddings model changes after some data already stored)
  - Partial inserts if a batch fails mid-run (depends on node implementation and Qdrant behavior)

#### A7) Move to Processed Folder
- **Type / role:** Google Drive (`n8n-nodes-base.googleDrive`) — moves file after successful vector insert.
- **Configuration:**
  - Operation: **move**
  - File ID: `={{ $('Download File').item.json.id }}`
  - Destination folder: ID `18ULhOgNCVhhbD0hvtCKBVoBGiCYQ8Dfp` (named “Qdrant готово”)
  - Drive: “My Drive”
- **Connections:**
  - Input: **Insert into Qdrant**
- **Edge cases / failures:**
  - If insert succeeds but move fails → file stays in incoming folder and may be reprocessed on next trigger (duplicate vectors)
  - Drive permissions / folder ID wrong

---

### Block B — Telegram Chat Flow (Trigger → Authorization → AI Agent → Reply)

**Overview:**  
This block receives Telegram messages, enforces an authorization check by chat ID, runs an AI agent that can retrieve context from Qdrant, and posts the final answer back into the same chat.

**Nodes involved:**
- Telegram Message Trigger
- Filter Authorized User
- AI Agent1
- OpenAI Chat Model1
- Qdrant Knowledge Base
- Send Response to Telegram

#### B1) Telegram Message Trigger
- **Type / role:** Telegram Trigger (`n8n-nodes-base.telegramTrigger`) — entry point for messages.
- **Configuration:**
  - Updates: `message`
- **Connections:**
  - Output → **Filter Authorized User**
- **Version notes:** TypeVersion `1.2`
- **Edge cases / failures:**
  - Telegram webhook misconfiguration or bot token invalid
  - If running behind firewalls/reverse proxies, webhook reachability issues

#### B2) Filter Authorized User
- **Type / role:** Filter (`n8n-nodes-base.filter`) — allowlist gate.
- **Configuration:**
  - Condition: `{{ $json.message.chat.id }}` **equals** `26899549`
  - Strict type validation enabled (node condition “typeValidation: strict”)
- **Connections:**
  - If passes → **AI Agent1**
  - If fails → no output (message is dropped)
- **Version notes:** TypeVersion `2.3`
- **Edge cases / failures:**
  - Group chats: `chat.id` differs from user id; you may lock yourself out unintentionally
  - If Telegram payload shape differs (non-text updates), `message.chat.id` may be missing → filter may error or evaluate false depending on node behavior

#### B3) AI Agent1
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates LLM reasoning + tool calls (Qdrant retrieval).
- **Configuration:**
  - Input text: `={{ $json.message.text }}`
  - Prompt type: **define**
  - System message: large style/persona prompt (expert in n8n automation, e-commerce, AI integration; requires using Qdrant tool before answering; Telegram formatting constraints)
  - Mentions: conversation memory “per chat” in sticky note, but **no dedicated memory node is present** in this JSON. Any memory behavior would have to be internal/default for this agent node/version or not actually implemented.
- **Connections:**
  - Main input: **Filter Authorized User**
  - AI language model input: from **OpenAI Chat Model1** via `ai_languageModel`
  - AI tool input: from **Qdrant Knowledge Base** via `ai_tool`
  - Main output → **Send Response to Telegram**
- **Version notes:** TypeVersion `3`
- **Edge cases / failures:**
  - If incoming message has no `text` (stickers, photos), `$json.message.text` is null → agent may error or respond oddly
  - Tool-call failures: Qdrant downtime, embedding mismatch, empty retrieval results
  - Very long user messages may exceed model context limits

#### B4) OpenAI Chat Model1
- **Type / role:** OpenAI Chat LLM (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — the model used by the agent.
- **Configuration:**
  - Model: `ft:gpt-4.1-2025-04-14:aimagine:adept3:CrV9Ir4p` (fine-tuned GPT‑4.1)
  - No built-in tools enabled here (agent uses separate tool wiring)
- **Connections:**
  - Output (`ai_languageModel`) → **AI Agent1**
- **Version notes:** TypeVersion `1.3`
- **Edge cases / failures:**
  - Fine-tuned model access restrictions, model not available in your OpenAI project
  - Higher cost/latency than smaller models

#### B5) Qdrant Knowledge Base
- **Type / role:** Qdrant Vector Store as Tool (`@n8n/n8n-nodes-langchain.vectorStoreQdrant`) — retrieval tool callable by agent.
- **Configuration:**
  - Mode: **retrieve-as-tool**
  - Tool description (Russian): “Используйте эту базу знаний, чтобы отвечать на вопросы пользователя” (Use this knowledge base to answer user questions)
  - Collection: `Test-youtube-adept-ecom`
  - `includeDocumentMetadata`: false
  - Requires embeddings input from **OpenAI Embeddings**
- **Connections:**
  - Input embeddings: **OpenAI Embeddings** (`ai_embedding`)
  - Output tool: (`ai_tool`) → **AI Agent1**
- **Version notes:** TypeVersion `1.3`
- **Edge cases / failures:**
  - If collection is empty, tool returns no context; agent must still answer
  - If embeddings model differs from what was used to insert, retrieval quality collapses

#### B6) Send Response to Telegram
- **Type / role:** Telegram send message (`n8n-nodes-base.telegram`) — posts final output.
- **Configuration:**
  - Chat ID: `={{ $('Telegram Message Trigger').item.json.message.chat.id }}`
  - Text: `={{ $json.output }}` (expects agent output field named `output`)
  - `appendAttribution`: false
- **Connections:**
  - Input: **AI Agent1**
- **Version notes:** TypeVersion `1.2`
- **Edge cases / failures:**
  - If AI Agent output field name differs (e.g., `text` instead of `output`), message will be blank/error
  - Telegram message length limits; long outputs may fail or truncate (depending on node behavior)

---

### Block C — Sticky Notes / Embedded Documentation

**Overview:**  
These nodes are visual documentation inside the canvas. They don’t execute logic but contain important setup instructions and context (including a YouTube link and security warning).

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

**Edge cases:** None (non-executing).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New File Trigger | googleDriveTrigger | Poll Google Drive folder for new files | — | Download File | ## Document Processing Flow<br>Monitors Google Drive → Downloads files → Splits text → Creates embeddings → Stores in Qdrant → Moves to processed folder |
| Download File | googleDrive | Download newly created file binary | New File Trigger | Insert into Qdrant | ## Document Processing Flow<br>Monitors Google Drive → Downloads files → Splits text → Creates embeddings → Stores in Qdrant → Moves to processed folder |
| Split Text into Chunks | langchain text splitter (recursive) | Split extracted text into overlapping chunks | — (LangChain wiring) | Load Document Data (ai_textSplitter) | ## Document Processing Flow<br>Monitors Google Drive → Downloads files → Splits text → Creates embeddings → Stores in Qdrant → Moves to processed folder |
| Load Document Data | langchain document loader | Convert binary into documents for vectorization | Split Text into Chunks (ai_textSplitter) | Insert into Qdrant (ai_document) | ## Document Processing Flow<br>Monitors Google Drive → Downloads files → Splits text → Creates embeddings → Stores in Qdrant → Moves to processed folder |
| OpenAI Embeddings | langchain OpenAI embeddings | Generate embeddings for insert + retrieval | — | Insert into Qdrant (ai_embedding), Qdrant Knowledge Base (ai_embedding) | ## AI Components<br>Fine-tuned GPT-4.1 model, conversation memory per chat, OpenAI embeddings, and Qdrant knowledge base retrieval tool |
| Insert into Qdrant | langchain Qdrant vector store | Insert chunk embeddings into Qdrant collection | Download File (main), Load Document Data (ai_document), OpenAI Embeddings (ai_embedding) | Move to Processed Folder | ## Document Processing Flow<br>Monitors Google Drive → Downloads files → Splits text → Creates embeddings → Stores in Qdrant → Moves to processed folder |
| Move to Processed Folder | googleDrive | Move processed file to “Ready” folder | Insert into Qdrant | — | ## Document Processing Flow<br>Monitors Google Drive → Downloads files → Splits text → Creates embeddings → Stores in Qdrant → Moves to processed folder |
| Telegram Message Trigger | telegramTrigger | Receive incoming Telegram messages | — | Filter Authorized User | ## Telegram Chat Flow<br>Receives messages → Filters authorized users → AI Agent with vector search → Generates response → Sends to Telegram |
| Filter Authorized User | filter | Allow only a specific chat/user ID | Telegram Message Trigger | AI Agent1 | ⚠️ Update this chat ID to your own Telegram user ID to authorize yourself. Find your ID by messaging @userinfobot |
| Qdrant Knowledge Base | langchain Qdrant vector store | Retrieval tool for agent (RAG) | OpenAI Embeddings (ai_embedding) | AI Agent1 (ai_tool) | ## AI Components<br>Fine-tuned GPT-4.1 model, conversation memory per chat, OpenAI embeddings, and Qdrant knowledge base retrieval tool |
| OpenAI Chat Model1 | langchain OpenAI chat model | LLM powering the agent (fine-tuned GPT‑4.1) | — | AI Agent1 (ai_languageModel) | ## AI Components<br>Fine-tuned GPT-4.1 model, conversation memory per chat, OpenAI embeddings, and Qdrant knowledge base retrieval tool |
| AI Agent1 | langchain agent | Generates response, calls Qdrant tool | Filter Authorized User; Qdrant Knowledge Base (ai_tool); OpenAI Chat Model1 (ai_languageModel) | Send Response to Telegram | ## Telegram Chat Flow<br>Receives messages → Filters authorized users → AI Agent with vector search → Generates response → Sends to Telegram |
| Send Response to Telegram | telegram | Send agent output back to chat | AI Agent1 | — | ## Telegram Chat Flow<br>Receives messages → Filters authorized users → AI Agent with vector search → Generates response → Sends to Telegram |
| Sticky Note | stickyNote | Canvas documentation | — | — | # How it works<br>This workflow creates an intelligent Telegram bot with a knowledge base powered by Qdrant vector database. It has two independent flows: … (setup steps + customization) |
| Sticky Note1 | stickyNote | Canvas label for ingestion flow | — | — | ## Document Processing Flow<br>Monitors Google Drive → Downloads files → Splits text → Creates embeddings → Stores in Qdrant → Moves to processed folder |
| Sticky Note2 | stickyNote | Canvas label for Telegram flow | — | — | ## Telegram Chat Flow<br>Receives messages → Filters authorized users → AI Agent with vector search → Generates response → Sends to Telegram |
| Sticky Note3 | stickyNote | Canvas label for AI components | — | — | ## AI Components<br>Fine-tuned GPT-4.1 model, conversation memory per chat, OpenAI embeddings, and Qdrant knowledge base retrieval tool |
| Sticky Note4 | stickyNote | Security warning about chat ID | — | — | ⚠️ Update this chat ID to your own Telegram user ID to authorize yourself. Find your ID by messaging @userinfobot |
| Sticky Note5 | stickyNote | Video link | — | — | @[youtube](7oDXXBYKKps) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. **Google Drive OAuth2** credential (for trigger + drive operations).
   2. **Qdrant API** credential (Qdrant URL + API key).
   3. **OpenAI API** credential (project key with access to embeddings + GPT‑4.1 or your chosen model).
   4. **Telegram API** credential (create bot with **@BotFather**, paste token into n8n Telegram credential).

2) **Prepare external services**
   1. In **Google Drive**, create two folders:
      - Incoming folder (watched)
      - Processed/Ready folder
      Copy both folder IDs.
   2. In **Qdrant**, create (or choose) a collection, e.g. `Test-youtube-adept-ecom`.

3) **Build Document Processing Flow**
   1. Add node **Google Drive Trigger** named **“New File Trigger”**
      - Event: `fileCreated`
      - Trigger on: `specificFolder`
      - Folder to watch: set incoming folder ID
      - Polling: every **15 minutes**
   2. Add node **Google Drive** named **“Download File”**
      - Operation: `download`
      - File ID: expression `{{ $json.id }}`
      - Connect: **New File Trigger → Download File**
   3. Add node **Qdrant Vector Store** named **“Insert into Qdrant”**
      - Mode: `insert`
      - Collection: select/create your collection
      - Embedding batch size: `100`
      - (Leave other options default unless you have metadata needs)
      - Connect: **Download File → Insert into Qdrant** (main connection)
   4. Add node **Recursive Character Text Splitter** named **“Split Text into Chunks”**
      - Chunk size: `3000`
      - Chunk overlap: `300`
   5. Add node **Document Default Data Loader** named **“Load Document Data”**
      - Data type: `binary`
      - Connect: **Split Text into Chunks** to **Load Document Data** using the **AI Text Splitter** connection type.
   6. Add node **OpenAI Embeddings** named **“OpenAI Embeddings”**
      - Select OpenAI credentials
      - (Optional) choose a specific embeddings model to keep dimensions stable over time
   7. Wire LangChain inputs for insertion:
      - **Load Document Data** → **Insert into Qdrant** via **AI Document** connection
      - **OpenAI Embeddings** → **Insert into Qdrant** via **AI Embedding** connection
   8. Add node **Google Drive** named **“Move to Processed Folder”**
      - Operation: `move`
      - File ID: expression `{{ $('Download File').item.json.id }}`
      - Destination folder: set processed/ready folder ID
      - Connect: **Insert into Qdrant → Move to Processed Folder**

4) **Build Telegram Chat Flow**
   1. Add node **Telegram Trigger** named **“Telegram Message Trigger”**
      - Updates: `message`
   2. Add node **Filter** named **“Filter Authorized User”**
      - Condition: `{{ $json.message.chat.id }}` equals your authorized ID
      - (Find your ID by messaging **@userinfobot** as noted)
      - Connect: **Telegram Message Trigger → Filter Authorized User**
   3. Add node **OpenAI Chat Model** named **“OpenAI Chat Model1”**
      - Model: either your fine-tuned GPT‑4.1 model ID or a standard model available to your account
   4. Add node **Qdrant Vector Store** named **“Qdrant Knowledge Base”**
      - Mode: `retrieve-as-tool`
      - Collection: same as ingestion
      - Tool description: describe when the agent should use it (as in the workflow)
      - Connect: **OpenAI Embeddings → Qdrant Knowledge Base** via **AI Embedding**
   5. Add node **AI Agent** named **“AI Agent1”**
      - Text: `{{ $json.message.text }}`
      - Prompt type: `define`
      - System message: paste/customize the persona + instructions (including “always use Qdrant tool before answering”)
      - Connect:
        - **Filter Authorized User → AI Agent1** (main)
        - **OpenAI Chat Model1 → AI Agent1** via **AI Language Model**
        - **Qdrant Knowledge Base → AI Agent1** via **AI Tool**
   6. Add node **Telegram** named **“Send Response to Telegram”**
      - Operation: send message
      - Chat ID: `{{ $('Telegram Message Trigger').item.json.message.chat.id }}`
      - Text: `{{ $json.output }}`
      - Connect: **AI Agent1 → Send Response to Telegram**

5) **Activate & test**
   - Upload a file into the incoming Drive folder; wait for polling; confirm vectors appear in Qdrant and file is moved.
   - Send a Telegram message from the authorized account; confirm the bot replies.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How it works + setup steps + customization guidance (Google Drive/Qdrant/OpenAI/Telegram) | Sticky note content embedded in the canvas (“How it works”) |
| ⚠️ Update authorized chat ID; use @userinfobot to find it | Security/access control note in canvas |
| @[youtube](7oDXXBYKKps) | Video link embedded in canvas |

