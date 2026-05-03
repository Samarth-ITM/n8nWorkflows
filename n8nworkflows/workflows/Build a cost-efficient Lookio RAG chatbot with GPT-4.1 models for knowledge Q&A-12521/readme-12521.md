Build a cost-efficient Lookio RAG chatbot with GPT-4.1 models for knowledge Q&A

https://n8nworkflows.xyz/workflows/build-a-cost-efficient-lookio-rag-chatbot-with-gpt-4-1-models-for-knowledge-q-a-12521


# Build a cost-efficient Lookio RAG chatbot with GPT-4.1 models for knowledge Q&A

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow implements a **cost-efficient RAG (Retrieval-Augmented Generation) chatbot** using an **intent router** approach. It uses a **very small/cheap model** for classification + simple replies, and only uses the **RAG + large model** path when the user’s question requires external knowledge from **Lookio**.

**Primary use cases:**
- Document/knowledge base Q&A backed by Lookio
- Mixed conversations (small talk + real questions) with minimized token cost
- Chat experiences requiring short-term conversational context (buffer memory)

### 1.1 Receive message & load conversation context
Receives a chat message, loads recent conversation history from memory, and provides it downstream for classification and prompt context.

### 1.2 Intent routing (cheap classification)
Uses a small OpenAI model to classify whether the user message requires knowledge retrieval.

### 1.3 Response generation (two branches)
- **1.3.1 Simple response branch:** small model writes the answer directly.
- **1.3.2 RAG branch:** mini model creates a retrieval query → HTTP call to Lookio → large model writes final answer from retrieved knowledge.

### 1.4 Response handling & memory persistence
Sends the answer back to chat and stores the user/AI messages into memory for future turns.

---

## 2. Block-by-Block Analysis

### Block 1 — Receive user message & load the conversation
**Overview:**  
Captures the incoming chat message and retrieves prior messages from a buffer-based memory so prompts can include conversation context.

**Nodes involved:**
- When chat message received
- Simple Memory
- Find past messages

#### Node: When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry point for chat-based workflows.
- **Key configuration:**
  - **Response mode:** `responseNodes` (the workflow responds via explicit “Respond to Chat” node(s), not directly in the trigger).
- **Inputs/Outputs:**
  - **Output →** `Find past messages` (main)
- **Edge cases / failures:**
  - If used outside n8n Chat UI context, `chatInput` may be missing.
  - Webhook configuration must remain consistent; changing it can break existing integrations.

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — windowed conversation memory store.
- **Key configuration:** default window settings (not explicitly set in parameters).
- **Connections:**
  - **ai_memory →** `Find past messages` and `Store messages`
- **Edge cases / failures:**
  - If the memory window is too small (default), older context may be lost, affecting query rewriting and final answers.

#### Node: Find past messages
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryManager` — reads messages from the configured memory.
- **Key configuration:**
  - Default mode here is effectively “read/get” (since `mode` isn’t set to insert).
- **Inputs/Outputs:**
  - **Input:** main from `When chat message received`
  - **Memory input:** from `Simple Memory` via `ai_memory`
  - **Output →** `Intent router` (main)
- **Key data used later:**
  - `$('Find past messages').item.json.messages` is referenced in multiple prompts via `.toJsonString()` or mapped formatting.
- **Edge cases / failures:**
  - If memory connection is missing/miswired, `messages` may be empty/undefined, breaking expressions in prompts (depending on node behavior).

**Sticky note(s) affecting this block:**
- “## 1. Receive user message & load the conversation”
- Also covered by the global note “Smart & Efficient RAG Chatbot …” (see Notes section).

---

### Block 2 — AI confirms if powerful RAG is needed (Intent Router)
**Overview:**  
Classifies the user message into “simple” vs “needs knowledge retrieval” using a very small model to reduce cost.

**Nodes involved:**
- Very small model
- Intent router

#### Node: Very small model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI chat model provider.
- **Model:** `gpt-4.1-nano`
- **Credentials:** OpenAI API credential named **“Duv's OpenAI”**
- **Connections:**
  - **ai_languageModel →** `Simple response` and `Intent router`
- **Edge cases / failures:**
  - Credential misconfiguration (invalid API key, missing org/project) causes auth errors.
  - Model availability depends on your OpenAI account/region; if unavailable, node fails.

#### Node: Intent router
- **Type / role:** `@n8n/n8n-nodes-langchain.textClassifier` — classifies text into provided categories.
- **Input text expression:**
  - `$('When chat message received').item.json.chatInput`
- **Categories:**
  1. “No knowledge retrieval needed to answer this query” (greetings, confirmations, small talk)
  2. “Knowledge retrieval is needed to answer this query” (questions requiring knowledge)
- **System prompt template (key detail):**
  - Includes previous messages as context:
    ```js
    $json.messages
      .map(m => `human: ${m.human}\nai: ${m.ai}`)
      .join('\n')
    ```
  - Instructs: “Don't explain, and only output the json.”
- **Routing / outputs:**
  - **Main output 0 →** `Simple response`
  - **Main output 1 →** `Prepare retrieval query`
  - (This implies the classifier returns two outputs based on the chosen category.)
- **Edge cases / failures:**
  - If `$json.messages` is undefined (memory read failed), the `.map(...)` expression can error.
  - Misclassification risk: a short question (“Price?”) might be routed incorrectly; consider adding examples or stricter category definitions if needed.
  - JSON-only output instruction: if the underlying model violates it, parsing may fail (depends on node implementation).

**Sticky note(s) affecting this block:**
- “## 2. AI confirms if powerful RAG is needed”
- Global “Smart & Efficient RAG Chatbot …”

---

### Block 3.1 — Writing a simple response (no RAG)
**Overview:**  
For low-effort queries, generates a direct response using a cheap model while still providing conversation context.

**Nodes involved:**
- Simple response
- Very small model (as provider)

#### Node: Simple response
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — LLM chain that composes prompt + user text.
- **Language model:** provided by **Very small model (gpt-4.1-nano)** via `ai_languageModel`.
- **User text:**
  - `$('When chat message received').item.json.chatInput`
- **System/content message (context injection):**
  - “You are a helpful assistant…”
  - Includes previous messages:
    - `{{ $('Find past messages').item.json.messages.toJsonString() }}`
- **Output / downstream:**
  - **Main →** `Respond to Chat`
- **Edge cases / failures:**
  - If `Find past messages` returns large history, prompts may get long; consider controlling memory window size.
  - If `messages.toJsonString()` is not available (depending on runtime/object shape), expression could fail; fallback would be `JSON.stringify(...)`.

**Sticky note(s) affecting this block:**
- “## 3.1. writing a simple response”

---

### Block 3.2 — Writing an AI knowledge retrieval based response (RAG path)
**Overview:**  
Transforms the user message into a concise retrieval query, sends it to Lookio for knowledge retrieval, then uses a larger model to craft the final answer grounded in retrieved content.

**Nodes involved:**
- Mini model
- Prepare retrieval query
- RAG via Lookio
- Large model
- Write the final response

#### Node: Mini model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI chat model provider.
- **Model:** `gpt-4.1-mini`
- **Credentials:** “Duv's OpenAI”
- **Connection:**
  - **ai_languageModel →** `Prepare retrieval query`
- **Edge cases / failures:**
  - Same as other OpenAI nodes: auth/model availability/rate limits.

#### Node: Prepare retrieval query
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — query rewriter for retrieval.
- **Language model:** from **Mini model (gpt-4.1-mini)**.
- **User text:**
  - `$('When chat message received').item.json.chatInput`
- **Prompt intent:**
  - Produce a “short and concise query” formulated as a question.
  - Includes conversation context:
    - `{{ $('Find past messages').item.json.messages.toJsonString() }}`
- **Output / downstream:**
  - **Main →** `RAG via Lookio`
  - The Lookio node uses `={{ $json.text }}` as the query, implying this node outputs `text`.
- **Edge cases / failures:**
  - The model may output extra text; consider adding stricter formatting (“Output only the question, no quotes”) if Lookio retrieval is sensitive.
  - If the chain outputs a different field than `text` (version-dependent), the next node’s expression will break.

#### Node: RAG via Lookio
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Lookio retrieval endpoint.
- **Request:**
  - **Method:** POST
  - **URL:** `https://api.lookio.app/webhook/query`
  - **Headers:**
    - `api_key: <YOUR-API-KEY>` (placeholder; must be replaced)
  - **Body parameters:**
    - `query: {{ $json.text }}`
    - `assistant_id: <YOUR-ASSISTANT-ID>` (placeholder; must be replaced)
    - `query_mode: flash`
- **Output / downstream:**
  - **Main →** `Write the final response`
  - `Write the final response` references `{{ $json.Output }}`, implying Lookio returns an `Output` field containing retrieved info.
- **Edge cases / failures:**
  - 401/403 if API key is wrong.
  - 400 if `assistant_id` is invalid or missing.
  - Response shape mismatch: if Lookio returns `output` (lowercase) or nested fields, the final prompt will be empty.
  - Network timeouts / rate limiting; consider retries/backoff in HTTP node options.

**Sub-workflow reference:** none (direct HTTP call).

#### Node: Large model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — higher-capability model for final grounded answer.
- **Model:** `gpt-4.1`
- **Credentials:** “Duv's OpenAI”
- **Connection:**
  - **ai_languageModel →** `Write the final response`
- **Edge cases / failures:**
  - Higher cost; ensure the router effectively gates usage.
  - Rate limits more likely under load.

#### Node: Write the final response
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — final answer generation from retrieved knowledge.
- **Language model:** from **Large model (gpt-4.1)**.
- **Input text (composed):**
  - Includes initial user query and retrieval output:
    - Initial query: `$('When chat message received').item.json.chatInput`
    - Knowledge retrieval output: `{{ $json.Output }}`
- **System/content message:**
  - Instructs that the message contains query + retrieval insights.
  - Includes prior messages:
    - `{{ $('Find past messages').item.json.messages.toJsonString() }}`
- **Output / downstream:**
  - **Main →** `Respond to Chat`
- **Edge cases / failures:**
  - Hallucination risk if retrieval output is empty/irrelevant; you may want to add an instruction like “If retrieval output is empty, ask a clarifying question.”
  - Token limits if Lookio returns very large text; consider truncation/summarization.

**Sticky note(s) affecting this block:**
- “## 3.2. Writing an AI knowledge retrieval based response”
- “## Action required: Make sure to set your Lookio API key and workspace ID in here.” (applies specifically to the Lookio HTTP node; note it says “workspace ID” but the node uses `assistant_id`)

---

### Block 4 — Response handling
**Overview:**  
Returns the AI response to the chat and saves the exchange (user + AI) into memory for future context.

**Nodes involved:**
- Respond to Chat
- Store messages
- Simple Memory (as memory provider)

#### Node: Respond to Chat
- **Type / role:** `@n8n/n8n-nodes-langchain.chat` — sends a message back to the chat UI/session.
- **Message expression:** `{{ $json.text }}` (expects upstream chain output in `text`)
- **Options:**
  - `memoryConnection: false` (memory is handled separately via Memory Manager)
  - `waitUserReply: false`
- **Inputs/Outputs:**
  - **Input:** from either `Simple response` or `Write the final response`
  - **Output →** `Store messages`
- **Edge cases / failures:**
  - If upstream node outputs a different field than `text`, the user will get an empty message.
  - If used outside a chat-enabled execution context, it may not deliver messages.

#### Node: Store messages
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryManager` — inserts messages into memory.
- **Mode:** `insert`
- **Messages stored:**
  - User: `$('When chat message received').item.json.chatInput`
  - AI: `{{ $json.text }}`
- **Memory connection:**
  - Receives `ai_memory` from `Simple Memory`
- **Inputs/Outputs:**
  - **Input:** main from `Respond to Chat`
  - **Output:** none (end)
- **Edge cases / failures:**
  - If memory connection missing, insert fails or silently does nothing (implementation-dependent).
  - If chat input is null/empty, you’ll store blank turns.

**Sticky note(s) affecting this block:**
- “## 4. Response handling”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entrypoint trigger | — | Find past messages | # **Smart & Efficient RAG Chatbot**… (includes Lookio + OpenAI steps, video link, credits) |
| Find past messages | @n8n/n8n-nodes-langchain.memoryManager | Read conversation history from memory | When chat message received; Simple Memory (ai_memory) | Intent router | ## 1. Receive user message & load the conversation |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Windowed chat memory store | — | Find past messages (ai_memory); Store messages (ai_memory) | ## 1. Receive user message & load the conversation |
| Very small model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Cheap LLM for routing + simple responses | — | Intent router (ai_languageModel); Simple response (ai_languageModel) | ## 2. AI confirms if powerful RAG is needed |
| Intent router | @n8n/n8n-nodes-langchain.textClassifier | Classify: RAG needed vs not | Find past messages | Simple response; Prepare retrieval query | ## 2. AI confirms if powerful RAG is needed |
| Simple response | @n8n/n8n-nodes-langchain.chainLlm | Generate direct answer without retrieval | Intent router; Very small model (ai_languageModel) | Respond to Chat | ## 3.1. writing a simple response |
| Mini model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Mid-size LLM to rewrite retrieval query | — | Prepare retrieval query (ai_languageModel) | ## 3.2. Writing an AI knowledge retrieval based response |
| Prepare retrieval query | @n8n/n8n-nodes-langchain.chainLlm | Convert user message into concise retrieval question | Intent router; Mini model (ai_languageModel) | RAG via Lookio | ## 3.2. Writing an AI knowledge retrieval based response |
| RAG via Lookio | n8n-nodes-base.httpRequest | Call Lookio API to retrieve relevant knowledge | Prepare retrieval query | Write the final response | ## Action required Make sure to set your Lookio API key and workspace ID in here. |
| Large model | @n8n/n8n-nodes-langchain.lmChatOpenAi | High-capability LLM for final grounded answer | — | Write the final response (ai_languageModel) | ## 3.2. Writing an AI knowledge retrieval based response |
| Write the final response | @n8n/n8n-nodes-langchain.chainLlm | Produce final answer using retrieval output | RAG via Lookio; Large model (ai_languageModel) | Respond to Chat | ## 3.2. Writing an AI knowledge retrieval based response |
| Respond to Chat | @n8n/n8n-nodes-langchain.chat | Send answer back to user | Simple response OR Write the final response | Store messages | ## 4. Response handling |
| Store messages | @n8n/n8n-nodes-langchain.memoryManager | Persist user + AI messages into memory | Respond to Chat; Simple Memory (ai_memory) | — | ## 4. Response handling |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/branding | — | — | |
| Sticky Note1 | n8n-nodes-base.stickyNote | Configuration reminder (Lookio placeholders) | — | — | |
| Sticky Note2 | n8n-nodes-base.stickyNote | Block label | — | — | |
| Sticky Note3 | n8n-nodes-base.stickyNote | Block label | — | — | |
| Sticky Note4 | n8n-nodes-base.stickyNote | Block label | — | — | |
| Sticky Note5 | n8n-nodes-base.stickyNote | Block label | — | — | |
| Sticky Note6 | n8n-nodes-base.stickyNote | Block label | — | — | |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1) Add node: **Chat Trigger** (`When chat message received`)  
   2) Set **Options → Response Mode** = `Response Nodes`.

2. **Add memory (window buffer)**
   1) Add node: **Memory Buffer Window** (`Simple Memory`)  
   2) Leave defaults (or configure window size if desired).

3. **Load past messages**
   1) Add node: **Memory Manager** (`Find past messages`)  
   2) Connect:
      - `When chat message received` → `Find past messages` (main)
      - `Simple Memory` → `Find past messages` (ai_memory)

4. **Create OpenAI model providers (3 nodes)**
   1) Add node: **OpenAI Chat Model** (`Very small model`) and select model `gpt-4.1-nano`  
   2) Add node: **OpenAI Chat Model** (`Mini model`) and select model `gpt-4.1-mini`  
   3) Add node: **OpenAI Chat Model** (`Large model`) and select model `gpt-4.1`  
   4) **Credentials:** configure OpenAI API credentials once and assign to all three model nodes.

5. **Intent classification**
   1) Add node: **Text Classifier** (`Intent router`)  
   2) Configure:
      - **Input Text** = `{{ $('When chat message received').item.json.chatInput }}`
      - Two categories:
        - “No knowledge retrieval needed to answer this query”
        - “Knowledge retrieval is needed to answer this query”
      - **System Prompt Template**: include prior messages (as in the workflow) and require JSON-only output.
   3) Connect:
      - `Find past messages` → `Intent router` (main)
      - `Very small model` → `Intent router` (ai_languageModel)

6. **Simple response branch**
   1) Add node: **LLM Chain** (`Simple response`)  
   2) Configure:
      - **Text** = `{{ $('When chat message received').item.json.chatInput }}`
      - **Prompt type:** “Define”
      - Add a system message including previous messages:  
        `{{ $('Find past messages').item.json.messages.toJsonString() }}`
   3) Connect:
      - `Intent router` output 0 → `Simple response` (main)
      - `Very small model` → `Simple response` (ai_languageModel)

7. **RAG branch: query preparation**
   1) Add node: **LLM Chain** (`Prepare retrieval query`)  
   2) Configure:
      - **Text** = `{{ $('When chat message received').item.json.chatInput }}`
      - Prompt instructing to output a short retrieval query as a question
      - Include previous messages via `Find past messages`
   3) Connect:
      - `Intent router` output 1 → `Prepare retrieval query` (main)
      - `Mini model` → `Prepare retrieval query` (ai_languageModel)

8. **RAG branch: Lookio retrieval**
   1) Add node: **HTTP Request** (`RAG via Lookio`)  
   2) Configure:
      - **Method:** POST  
      - **URL:** `https://api.lookio.app/webhook/query`
      - **Send Headers:** on  
        - `api_key` = your Lookio API key
      - **Send Body:** on (Body Parameters)
        - `query` = `{{ $json.text }}`
        - `assistant_id` = your Lookio Assistant ID
        - `query_mode` = `flash`
   3) Connect:
      - `Prepare retrieval query` → `RAG via Lookio` (main)

9. **RAG branch: final answer**
   1) Add node: **LLM Chain** (`Write the final response`)  
   2) Configure:
      - **Text** combining:
        - user query: `{{ $('When chat message received').item.json.chatInput }}`
        - retrieval output: `{{ $json.Output }}`
      - System message explaining to answer using retrieval insights and include prior messages.
   3) Connect:
      - `RAG via Lookio` → `Write the final response` (main)
      - `Large model` → `Write the final response` (ai_languageModel)

10. **Respond back to chat**
   1) Add node: **Respond to Chat** (`Respond to Chat`)  
   2) Configure **Message** = `{{ $json.text }}`  
   3) Connect:
      - `Simple response` → `Respond to Chat` (main)
      - `Write the final response` → `Respond to Chat` (main)

11. **Store conversation messages**
   1) Add node: **Memory Manager** (`Store messages`)  
   2) Set **Mode** = `insert`  
   3) Add two message entries:
      - type `user`: `{{ $('When chat message received').item.json.chatInput }}`
      - type `ai`: `{{ $json.text }}`
   4) Connect:
      - `Respond to Chat` → `Store messages` (main)
      - `Simple Memory` → `Store messages` (ai_memory)

12. **(Optional) Add sticky notes**
   - Add the block-label notes and the Lookio configuration reminder for maintainability.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Smart & Efficient RAG Chatbot… routes simple chats to small models… Lookio… Connect AI… Configure Lookio… Test…” | Sticky note content from the workflow |
| Lookio website link | https://www.lookio.app/ |
| Video guide link (modular agent logic) | https://www.youtube.com/watch?v=BHdJFnx2wrc |
| Template credit: “A template created by Guillaume Duvernay” | Included in sticky note |
| “Action required: Make sure to set your Lookio API key and workspace ID in here.” | Applies to the `RAG via Lookio` HTTP Request node (placeholders must be replaced).