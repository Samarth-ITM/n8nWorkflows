Ask questions about past meetings using voice with OpenAI and Pinecone

https://n8nworkflows.xyz/workflows/ask-questions-about-past-meetings-using-voice-with-openai-and-pinecone-12757


# Ask questions about past meetings using voice with OpenAI and Pinecone

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Ask questions about past meetings using voice with OpenAI and Pinecone  
**Workflow name (in JSON):** Ask your meetings questions using voice and AI

This workflow implements a **voice-driven Q&A assistant for past meetings**. A user sends an **audio question** to a webhook; the workflow **transcribes** it, **retrieves relevant meeting notes from Pinecone via vector search**, then uses an **Azure OpenAI chat model** to generate a **short factual answer grounded only in retrieved context**, converts that answer back into **audio speech**, and returns it in the webhook response.

### Logical blocks
1. **1.1 Input Reception (Webhook)** – Receives the audio question.
2. **1.2 Voice-to-Text (OpenAI Transcription + Validation)** – Transcribes audio to text and extracts a safe `question` field.
3. **1.3 Vector Search (Embeddings + Pinecone retrieval)** – Creates embeddings and searches Pinecone in a team-based namespace.
4. **1.4 Context Assembly (RAG context building)** – Merges multiple retrieved documents into one context string.
5. **1.5 Answer Generation (Azure OpenAI Agent)** – Produces an answer constrained to the meeting context.
6. **1.6 Text-to-Speech and Response** – Converts the answer text to audio and returns it.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception (Webhook)

**Overview:** Accepts the user’s voice question via HTTP POST and defers the HTTP response until the end of the workflow using a response node.

**Nodes involved:**
- Receive Voice Question (Webhook)

#### Node: Receive Voice Question
- **Type / role:** `n8n-nodes-base.webhook` – entry point, receives inbound HTTP request.
- **Key configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `meeting-sentiment-query`
  - **Response Mode:** `responseNode` (the workflow must end with *Respond to Webhook* to reply)
  - **Options:** `binaryData: false`
- **Inputs / outputs:**
  - **Input:** external HTTP request
  - **Output:** to **Transcribe Voice Question**
- **Important edge cases / failures:**
  - **Binary audio handling mismatch:** transcription node expects a binary property `file`, but the webhook has `binaryData: false`. Unless the client sends audio in a way that n8n still maps to a binary field, transcription will fail with “binary property not found”.
  - Missing/incorrect content-type or multipart form field naming may prevent audio from being accessible as binary.

---

### 1.2 Voice-to-Text (OpenAI Transcription + Validation)

**Overview:** Transcribes incoming audio into text and normalizes it into `{ question, team }`, enforcing that transcription succeeded.

**Nodes involved:**
- Transcribe Voice Question
- Extract Voice Question

#### Node: Transcribe Voice Question
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` – OpenAI audio transcription.
- **Key configuration (interpreted):**
  - **Resource:** Audio
  - **Operation:** Transcribe
  - **Binary Property Name:** `file` (expects incoming binary audio at `$binary.file`)
- **Credentials:** OpenAI API credential (“OpenAi account 2”)
- **Inputs / outputs:**
  - **Input:** from **Receive Voice Question**
  - **Output:** to **Extract Voice Question**, produces JSON including `text` (the transcription result).
- **Edge cases / failures:**
  - Missing `$binary.file` (common if webhook not configured for binary/multipart)
  - Unsupported audio format, size limits, or OpenAI API auth/quota errors
  - Timeouts on large audio uploads

#### Node: Extract Voice Question
- **Type / role:** `n8n-nodes-base.code` – validates and reshapes data for downstream steps.
- **Key configuration (interpreted):**
  - Reads `$json.text` from transcription output.
  - Produces:
    - `question`: transcription text
    - `team`: `$json.team` if present, else default `'Test Team'`
- **Code behavior (key logic):**
  - Throws if `$json.text` missing or not a string:
    - `throw new Error('Voice question text not found');`
- **Inputs / outputs:**
  - **Input:** from **Transcribe Voice Question**
  - **Output:** to **Search Meetings in Pinecone**
- **Edge cases / failures:**
  - If transcription returns an unexpected structure (no `text`)
  - The default `team` may cause all queries to go to `'Test Team'` namespace unless the request provides a `team` field upstream

---

### 1.3 Vector Search (Embeddings + Pinecone retrieval)

**Overview:** Builds an embedding for the user question (OpenAI embeddings node) and uses it to query Pinecone for semantically relevant meeting notes in a team-scoped namespace.

**Nodes involved:**
- Generate Question Embedding
- Search Meetings in Pinecone

#### Node: Generate Question Embedding
- **Type / role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi` – produces an embedding vector for the question.
- **Key configuration (interpreted):**
  - Uses OpenAI embeddings with default options (no explicit model set in JSON).
- **Credentials:** OpenAI API credential (“OpenAi account 2”)
- **Inputs / outputs:**
  - **Input:** (implicitly the current item context; should contain the text to embed—typically the node embeds a field from the input depending on node defaults)
  - **Output (special):** connected via `ai_embedding` to **Search Meetings in Pinecone**
- **Edge cases / failures:**
  - If the embeddings node cannot determine which text to embed (depends on node defaults/version)
  - OpenAI auth/quota errors

#### Node: Search Meetings in Pinecone
- **Type / role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone` – vector DB retrieval.
- **Key configuration (interpreted):**
  - **Mode:** Load (retrieve)
  - **Index:** `meeting-sentiment-index`
  - **Namespace:** `{{ $json.team }}` (team-scoped isolation)
  - Receives embeddings via the `ai_embedding` connection
  - Also configured with:
    - `prompt: {{ $json.team }}` (unusual: “prompt” is set to the team name; likely intended for query text but embeddings are supplied separately)
- **Credentials:** Pinecone API credential (“PineconeApi account 2”)
- **Inputs / outputs:**
  - **Main input:** from **Extract Voice Question**
  - **Embedding input:** from **Generate Question Embedding** via `ai_embedding`
  - **Main output:** to **Build Meeting Context**, returns one item per retrieved document (commonly as `json.document.pageContent`)
- **Edge cases / failures:**
  - Namespace/index mismatch (empty results)
  - Pinecone auth/environment errors
  - If the node expects a query string but only embeddings are provided (or vice versa), retrieval may be poor or fail depending on node version/settings

---

### 1.4 Context Assembly (RAG context building)

**Overview:** Concatenates retrieved meeting documents into a single context string, separated by delimiters, for the answering agent.

**Nodes involved:**
- Build Meeting Context

#### Node: Build Meeting Context
- **Type / role:** `n8n-nodes-base.code` – transforms Pinecone results into a single `context` field.
- **Key configuration (interpreted):**
  - Reads all incoming items (`$input.all()`).
  - Extracts `item.json.document?.pageContent`.
  - Joins non-empty contents with `\n---\n`.
  - Outputs `{ context }`.
- **Failure behavior:**
  - If no items returned: `throw new Error('No documents returned from Pinecone');`
- **Inputs / outputs:**
  - **Input:** from **Search Meetings in Pinecone**
  - **Output:** to **Answer Question from Meetings**
- **Edge cases / failures:**
  - Pinecone items not shaped as `json.document.pageContent` → context becomes empty and may produce an empty output, or throw if no items
  - Very large combined context may exceed model context window or increase costs/latency

---

### 1.5 Answer Generation (Azure OpenAI Agent)

**Overview:** Uses an AI agent to answer the user question, constrained to the retrieved meeting context, powered by Azure OpenAI chat model.

**Nodes involved:**
- AI Chat Model
- Answer Question from Meetings

#### Node: AI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` – provides the chat LLM backend to the agent.
- **Key configuration (interpreted):**
  - **Model:** `GPT-4O-MINI`
- **Credentials:** Azure OpenAI credential (“Azure Open AI account”)
- **Connections:**
  - Feeds **Answer Question from Meetings** via `ai_languageModel`
- **Edge cases / failures:**
  - Azure deployment/model name mismatch (Azure often requires deployment name mapping)
  - Auth / rate limits / content filter responses

#### Node: Answer Question from Meetings
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` – generates an answer using the provided prompt and attached chat model.
- **Key configuration (interpreted):**
  - **Prompt type:** Define (explicit prompt text)
  - Prompt instructs:
    - “Answer using ONLY provided meeting context”
    - If not found: “say you don’t have that information”
    - Short and clear
  - Injected variables:
    - `{{ $json.context }}` from **Build Meeting Context**
    - `{{ $('Extract Voice Question').item.json.question }}` referencing the earlier node’s output
- **Inputs / outputs:**
  - **Main input:** from **Build Meeting Context**
  - **Model input:** from **AI Chat Model** via `ai_languageModel`
  - **Main output:** to **Prepare Answer for Audio**
  - Output commonly includes `json.output` (the agent’s generated text), which is later used for TTS.
- **Edge cases / failures:**
  - If expression `$('Extract Voice Question').item.json.question` can’t be resolved (e.g., execution data missing, multiple items mismatched), prompt may be broken.
  - If `context` is empty/too large, answer may be poor or fail.
  - Ensure the agent output field matches what downstream expects (`$json.output` is assumed later).

---

### 1.6 Text-to-Speech and Response

**Overview:** Takes the AI’s text output, converts it to audio speech using OpenAI audio generation, then returns the audio via webhook response.

**Nodes involved:**
- Prepare Answer for Audio
- Generate Spoken Answer
- Return Audio Response

#### Node: Prepare Answer for Audio
- **Type / role:** `n8n-nodes-base.code` – currently a placeholder transform.
- **Key configuration (interpreted):**
  - Loops all items and adds `item.json.myNewField = 1`.
  - Does not reshape/validate the actual answer text.
- **Inputs / outputs:**
  - **Input:** from **Answer Question from Meetings**
  - **Output:** to **Generate Spoken Answer**
- **Edge cases / failures:**
  - If downstream expects a specific field preparation (e.g., ensuring `$json.output` exists), this node currently does not guarantee it.
  - Consider replacing with explicit mapping/validation of the answer text.

#### Node: Generate Spoken Answer
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` – OpenAI audio generation (text-to-speech).
- **Key configuration (interpreted):**
  - **Resource:** Audio
  - **Input text:** `{{ $json.output }}` (expects the agent text in `output`)
- **Credentials:** OpenAI API credential (“OpenAi account 2”)
- **Inputs / outputs:**
  - **Input:** from **Prepare Answer for Audio**
  - **Output:** to **Return Audio Response** (likely returns binary audio data plus metadata)
- **Edge cases / failures:**
  - If `$json.output` is missing or not a string, TTS may fail.
  - OpenAI API limits/auth errors; audio format defaults may need to be set depending on client expectations.

#### Node: Return Audio Response
- **Type / role:** `n8n-nodes-base.respondToWebhook` – sends final HTTP response for webhook.
- **Key configuration (interpreted):**
  - Uses default options (no explicit headers/body mapping shown).
- **Inputs / outputs:**
  - **Input:** from **Generate Spoken Answer**
  - **Output:** HTTP response to original requester
- **Edge cases / failures:**
  - If audio is in binary, you may need to configure response headers/content type and ensure respond node returns binary correctly (depends on how the OpenAI audio node outputs data and how Respond to Webhook is configured in your n8n version).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / section label |  |  | ### Voice Input Handling  This workflow starts by receiveing an audio file containing the user’s spoken question and then it converts the audio question into readable text using OpenAI and extracts the spoken question text safely. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / section label |  |  | ### Vector Search (RAG)  Converts the question into a vector for semantic search and searches meeting data using the team namespace then combines multiple Pinecone results into one clean context block. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / section label |  |  | ### AI Answer Generation  The ai agent answers the question using only the retrieved meeting context and generates a short, clear, factual response. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / section label |  |  | ### Voice Response Output  Converts the AI’s text answer into spoken audio and sends the audio answer back to the user. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / workflow overview |  |  | ## Workflow Overview  This is a voice-activated meeting assistant workflow. It allows users to ask questions using their voice. The workflow searches past meeting notes stored in Pinecone and replies with a spoken answer.  ### How it works  The user sends a voice question to the webhook. The audio is converted into text using speech transcription by Open AI. The question text is cleaned and prepared for searching. Pinecone is then used to search relevant meeting notes. The AI generates an answer using only the retrieved meeting context. The answer is converted into speech. Finally, the audio response is sent back to the user.  ### Setup  OpenAI API (for transcription, embeddings, speech) Azure OpenAI (for chat responses) Pinecone API (vector database access) |
| Receive Voice Question | n8n-nodes-base.webhook | Receive audio question | External HTTP caller | Transcribe Voice Question | ### Voice Input Handling  This workflow starts by receiveing an audio file containing the user’s spoken question and then it converts the audio question into readable text using OpenAI and extracts the spoken question text safely. |
| Transcribe Voice Question | @n8n/n8n-nodes-langchain.openAi | Speech-to-text transcription | Receive Voice Question | Extract Voice Question | ### Voice Input Handling  This workflow starts by receiveing an audio file containing the user’s spoken question and then it converts the audio question into readable text using OpenAI and extracts the spoken question text safely. |
| Extract Voice Question | n8n-nodes-base.code | Validate and normalize `{question, team}` | Transcribe Voice Question | Search Meetings in Pinecone | ### Voice Input Handling  This workflow starts by receiveing an audio file containing the user’s spoken question and then it converts the audio question into readable text using OpenAI and extracts the spoken question text safely. |
| Generate Question Embedding | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Create embedding for semantic search | (Execution context) | Search Meetings in Pinecone (ai_embedding) | ### Vector Search (RAG)  Converts the question into a vector for semantic search and searches meeting data using the team namespace then combines multiple Pinecone results into one clean context block. |
| Search Meetings in Pinecone | @n8n/n8n-nodes-langchain.vectorStorePinecone | Retrieve meeting notes via Pinecone vector search | Extract Voice Question; Generate Question Embedding (ai_embedding) | Build Meeting Context | ### Vector Search (RAG)  Converts the question into a vector for semantic search and searches meeting data using the team namespace then combines multiple Pinecone results into one clean context block. |
| Build Meeting Context | n8n-nodes-base.code | Merge retrieved documents into one context | Search Meetings in Pinecone | Answer Question from Meetings | ### Vector Search (RAG)  Converts the question into a vector for semantic search and searches meeting data using the team namespace then combines multiple Pinecone results into one clean context block. |
| AI Chat Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI chat model backend | (None) | Answer Question from Meetings (ai_languageModel) | ### AI Answer Generation  The ai agent answers the question using only the retrieved meeting context and generates a short, clear, factual response. |
| Answer Question from Meetings | @n8n/n8n-nodes-langchain.agent | Generate grounded answer from context | Build Meeting Context; AI Chat Model (ai_languageModel) | Prepare Answer for Audio | ### AI Answer Generation  The ai agent answers the question using only the retrieved meeting context and generates a short, clear, factual response. |
| Prepare Answer for Audio | n8n-nodes-base.code | Placeholder: minor JSON modification | Answer Question from Meetings | Generate Spoken Answer | ### Voice Response Output  Converts the AI’s text answer into spoken audio and sends the audio answer back to the user. |
| Generate Spoken Answer | @n8n/n8n-nodes-langchain.openAi | Text-to-speech | Prepare Answer for Audio | Return Audio Response | ### Voice Response Output  Converts the AI’s text answer into spoken audio and sends the audio answer back to the user. |
| Return Audio Response | n8n-nodes-base.respondToWebhook | Return final audio to caller | Generate Spoken Answer | (HTTP response) | ### Voice Response Output  Converts the AI’s text answer into spoken audio and sends the audio answer back to the user. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Ask your meetings questions using voice and AI”** (or your preferred name).

2. **Add Webhook node**
   - Node type: **Webhook**
   - Name: **Receive Voice Question**
   - Method: **POST**
   - Path: **meeting-sentiment-query**
   - Response mode: **Using “Respond to Webhook” node**
   - Important: configure request handling so the uploaded audio is available as **binary property `file`** (commonly via multipart/form-data with a file field named `file`, and webhook set to accept binary data).

3. **Add OpenAI Transcription node**
   - Node type: **OpenAI (LangChain)**
   - Name: **Transcribe Voice Question**
   - Resource: **Audio**
   - Operation: **Transcribe**
   - Binary property name: **file**
   - Credentials: configure **OpenAI API key** credential and select it in the node.

4. **Add Code node to extract/validate question**
   - Node type: **Code**
   - Name: **Extract Voice Question**
   - Paste logic (conceptually):
     - Validate `text` exists and is a string.
     - Output JSON: `{ question: text, team: incomingTeamOrDefault }`
   - Ensure it outputs:
     - `question` (string)
     - `team` (string, default “Test Team”)

5. **Add OpenAI Embeddings node**
   - Node type: **OpenAI Embeddings (LangChain)**
   - Name: **Generate Question Embedding**
   - Credentials: same **OpenAI API** credential
   - Configure the node so it embeds the **question text** (depending on node UI/version, set the input text field to `{{$json.question}}` if required).

6. **Add Pinecone Vector Store node**
   - Node type: **Pinecone Vector Store (LangChain)**
   - Name: **Search Meetings in Pinecone**
   - Mode: **Load / Search**
   - Index: **meeting-sentiment-index**
   - Namespace: `{{$json.team}}`
   - Ensure the node receives:
     - **Main input** from **Extract Voice Question**
     - **Embedding input** from **Generate Question Embedding** (connect the embeddings node to the Pinecone node’s **embedding** connector)
   - Credentials: configure **Pinecone API key** credential.

7. **Add Code node to build context**
   - Node type: **Code**
   - Name: **Build Meeting Context**
   - Logic:
     - Take all returned documents
     - Extract `document.pageContent`
     - Join with `---`
     - Output `{ context: "...combined..." }`
   - Optionally, instead of throwing on empty results, you can output an empty context and let the agent respond “I don’t have that information.”

8. **Add Azure OpenAI Chat Model node**
   - Node type: **Azure OpenAI Chat Model (LangChain)**
   - Name: **AI Chat Model**
   - Model/deployment: set to **GPT-4O-MINI** (must match your Azure OpenAI deployment configuration)
   - Credentials: configure **Azure OpenAI** credential (endpoint, key, deployment/model mapping).

9. **Add AI Agent node**
   - Node type: **AI Agent (LangChain)**
   - Name: **Answer Question from Meetings**
   - Prompt type: **Define**
   - Prompt content should:
     - Restrict answers to `context`
     - Include `{{$json.context}}`
     - Include the user question from the earlier node, e.g. `{{$('Extract Voice Question').item.json.question}}`
   - Connect **AI Chat Model** to the agent via the **Language Model** connector.
   - Connect **Build Meeting Context** main output to the agent main input.

10. **Add Code node (optional preparation)**
   - Node type: **Code**
   - Name: **Prepare Answer for Audio**
   - In the provided workflow, it only adds a dummy field. Prefer to ensure:
     - `answerText: $json.output` (or similar)
     - Validate non-empty string before TTS.

11. **Add OpenAI Text-to-Speech node**
   - Node type: **OpenAI (LangChain)**
   - Name: **Generate Spoken Answer**
   - Resource: **Audio** (TTS)
   - Input text: `{{$json.output}}` (or your mapped answer text field)
   - Credentials: **OpenAI API**
   - Configure voice/format options if needed (not specified in the JSON).

12. **Add Respond to Webhook node**
   - Node type: **Respond to Webhook**
   - Name: **Return Audio Response**
   - Configure to return the audio produced by the previous node (binary/stream) with correct headers if your client requires them.

13. **Connect nodes in order**
   - Receive Voice Question → Transcribe Voice Question → Extract Voice Question → Search Meetings in Pinecone → Build Meeting Context → Answer Question from Meetings → Prepare Answer for Audio → Generate Spoken Answer → Return Audio Response
   - Generate Question Embedding → (embedding connector) → Search Meetings in Pinecone
   - AI Chat Model → (ai_languageModel connector) → Answer Question from Meetings

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI API is used for transcription, embeddings, and speech generation. Azure OpenAI is used for chat responses. Pinecone is used as the vector database. | From the workflow overview sticky note (“Setup” section). |
| Team-based retrieval is implemented via `pineconeNamespace = {{$json.team}}`; if `team` is not supplied, it defaults to `Test Team`. | Extract Voice Question + Pinecone node configuration. |
| Potential configuration issue: the webhook has `binaryData: false` while transcription expects binary property `file`. | Ensure multipart/binary ingestion is correctly configured in n8n and by the client. |

