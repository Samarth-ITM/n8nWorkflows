Guide students with an AI tutor chatbot using RAG, intent classification and GPT-4o-mini

https://n8nworkflows.xyz/workflows/guide-students-with-an-ai-tutor-chatbot-using-rag--intent-classification-and-gpt-4o-mini-14291


# Guide students with an AI tutor chatbot using RAG, intent classification and GPT-4o-mini

# 1. Workflow Overview

This workflow implements **Nathan**, an AI tutor chatbot for guided educational conversations. It receives a message from a public chat interface, classifies the learner’s intent, fetches a matching instructional system prompt from Google Sheets, generates a response with **GPT-4o-mini** enhanced by **Pinecone RAG**, logs the exchange, and returns the formatted reply to the chat UI.

Typical use cases include:

- welcoming learners and starting topic-based sessions
- distinguishing between greetings, topic requests, answers, questions, and off-topic messages
- grounding responses in a book-based vector knowledge base
- preserving chat history for review, quality control, or auditing

The workflow is organized into four main logical blocks.

## 1.1 Input Reception & Intent Classification

The public chat trigger receives user input and sends it to a DeepSeek-backed classifier agent. A short-term memory buffer supports contextual interpretation of follow-up messages.

## 1.2 Prompt Retrieval & Input Aggregation

The classifier output is used to query a Google Sheets prompt library. The selected prompt is then merged with the original user input and aggregated into a single payload for the teaching agent.

## 1.3 AI Teacher Agent with RAG

A second agent, powered by GPT-4o-mini, uses the fetched system prompt, session memory, and Pinecone vector retrieval tool to generate a pedagogically relevant response.

## 1.4 Logging & Response Return

The generated answer is appended to a Google Sheets preservation log, then formatted into the expected output structure for the chat frontend.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Input Reception & Intent Classification

### Overview

This block receives the user message from the public chat interface and determines the learner’s intent category. It uses a DeepSeek language model plus memory to classify each message into one of five labels: `greeting`, `topic`, `answer`, `question`, or `random`.

### Nodes Involved

- Chat Trigger
- Intent Classifier
- DeepSeek LLM
- Classifier Memory

### Node Details

#### Chat Trigger

- **Type and role:** `@n8n/n8n-nodes-langchain.chatTrigger`; public conversational entry point for the chatbot.
- **Configuration choices:**
  - Public chat endpoint enabled
  - Initial greeting message configured as:
    - “Hi there! 👋”
    - “My name is Nathan. Please share the topic you want to learn today!”
- **Key expressions or variables used:**
  - Produces `chatInput`
  - Produces `sessionId`
- **Input and output connections:**
  - No upstream node; this is an entry point
  - Sends output to:
    - `Merge Input and Prompt`
    - `Intent Classifier`
- **Version-specific requirements:**
  - Uses node type version `1.1`
  - Requires n8n installation supporting LangChain chat trigger nodes
- **Edge cases / failures:**
  - Public exposure can invite spam or irrelevant traffic
  - If chat frontend or webhook routing is misconfigured, messages may never arrive
  - If downstream nodes assume `chatInput` exists, malformed trigger payloads can break expressions
- **Sub-workflow reference:** None

#### Intent Classifier

- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; LLM-based classification agent that maps incoming chat to one of five intents.
- **Configuration choices:**
  - Prompt source is defined directly in the node
  - User text passed as `={{ $json.chatInput }}`
  - System message strictly instructs the model to return only one word:
    - `greeting`
    - `topic`
    - `answer`
    - `question`
    - `random`
  - The prompt also biases the model toward `question` when memory suggests an ongoing conversation
- **Key expressions or variables used:**
  - `={{ $json.chatInput }}`
- **Input and output connections:**
  - Main input from `Chat Trigger`
  - Receives language model input from `DeepSeek LLM`
  - Receives memory input from `Classifier Memory`
  - Main output to `Fetch System Prompt`
- **Version-specific requirements:**
  - Uses node type version `1.9`
  - Requires compatible LangChain agent support in n8n
- **Edge cases / failures:**
  - The classifier may output unexpected formatting if the model ignores instructions
  - Misclassification can route the learner to the wrong prompt
  - If `chatInput` is empty, classification quality collapses
  - If memory context grows inconsistent, follow-up answers may be wrongly treated as questions
- **Sub-workflow reference:** None

#### DeepSeek LLM

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatDeepSeek`; language model backend for the intent classifier.
- **Configuration choices:**
  - Minimal configuration; default options used
  - Credentials must be added manually
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Intent Classifier` through `ai_languageModel`
- **Version-specific requirements:**
  - Uses node type version `1`
  - Requires available DeepSeek credentials and node support in the installed n8n version
- **Edge cases / failures:**
  - Invalid API key or quota exhaustion
  - Model unavailability or provider latency
  - Output drift from expected single-token classification format
- **Sub-workflow reference:** None

#### Classifier Memory

- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; conversational memory for the classifier.
- **Configuration choices:**
  - No custom parameters visible; defaults are used
- **Key expressions or variables used:** None explicitly configured
- **Input and output connections:**
  - Connected to `Intent Classifier` through `ai_memory`
- **Version-specific requirements:**
  - Uses node type version `1.3`
- **Edge cases / failures:**
  - Default memory behavior may not align with desired session scoping
  - If memory is not tied to session reliably, multiple chats may contaminate each other depending on runtime configuration
- **Sub-workflow reference:** None

---

## 2.2 Block: Prompt Retrieval & Input Aggregation

### Overview

This block turns the classification result into a system prompt lookup in Google Sheets. It then merges the original chat input with the retrieved prompt so the teacher agent can receive both together.

### Nodes Involved

- Fetch System Prompt
- Merge Input and Prompt
- Aggregate Fields

### Node Details

#### Fetch System Prompt

- **Type and role:** `n8n-nodes-base.googleSheets`; fetches the system prompt matching the classified intent from the prompt library.
- **Configuration choices:**
  - Google Sheets document ID is currently a placeholder: `YOUR_GOOGLE_SHEET_ID`
  - Sheet selected: `pmt` tab (`gid=0`)
  - Lookup filter:
    - `lookupColumn`: `Output`
    - `lookupValue`: `={{ $json.output }}`
  - This implies the classifier result must appear in the `Output` column of the prompt sheet
- **Key expressions or variables used:**
  - `={{ $json.output }}`
- **Input and output connections:**
  - Main input from `Intent Classifier`
  - Main output to `Merge Input and Prompt` input 1
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.5`
  - Requires valid Google Sheets credentials
- **Edge cases / failures:**
  - Placeholder Sheet ID must be replaced
  - If the sheet tab or `Output` column does not exist, lookup fails
  - If the classifier returns unexpected casing or spacing, no row may match
  - Multiple matching rows may lead to ambiguous downstream data
  - Permission errors if the configured Google account cannot access the document
- **Sub-workflow reference:** None

#### Merge Input and Prompt

- **Type and role:** `n8n-nodes-base.merge`; joins original trigger data with the prompt lookup result.
- **Configuration choices:**
  - Default merge behavior is used
  - Receives:
    - original chat payload on input 0
    - fetched prompt row on input 1
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input 0 from `Chat Trigger`
  - Input 1 from `Fetch System Prompt`
  - Output to `Aggregate Fields`
- **Version-specific requirements:**
  - Uses node type version `3.1`
- **Edge cases / failures:**
  - If one branch returns zero items, merge behavior may produce no output depending on runtime semantics
  - If fetched prompt data has multiple rows, merge may duplicate chat items
  - Field name collisions can overwrite data
- **Sub-workflow reference:** None

#### Aggregate Fields

- **Type and role:** `n8n-nodes-base.aggregate`; consolidates the merged data into array fields expected downstream.
- **Configuration choices:**
  - Aggregates:
    - `Prompt`
    - `chatInput`
  - Output is grouped into aggregated collections rather than passing through raw rows
- **Key expressions or variables used:** None directly, but downstream relies on:
  - `Prompt`
  - `chatInput`
- **Input and output connections:**
  - Input from `Merge Input and Prompt`
  - Output to `AI Teacher Agent`
- **Version-specific requirements:**
  - Uses node type version `1`
- **Edge cases / failures:**
  - If the Google Sheets result does not contain a `Prompt` field, downstream system message expression fails
  - Aggregation into arrays means downstream expressions must reference array indices correctly
  - Empty aggregation can produce malformed prompt payloads
- **Sub-workflow reference:** None

---

## 2.3 Block: AI Teacher Agent with RAG

### Overview

This block generates the actual tutoring response. It uses GPT-4o-mini as the chat model, a session-scoped memory buffer for continuity, and a Pinecone vector store tool with OpenAI embeddings for retrieval of relevant educational content.

### Nodes Involved

- AI Teacher Agent
- GPT-4o-mini LLM
- Teacher Session Memory
- Book Knowledge Base
- OpenAI Embeddings

### Node Details

#### AI Teacher Agent

- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; primary teaching agent that answers learner messages using dynamic prompting and retrieval.
- **Configuration choices:**
  - User text is pulled directly from the chat trigger:
    - `={{ $('Chat Trigger').item.json.chatInput }}`
  - System prompt is dynamic:
    - `={{ $json.Prompt[0] }}`
  - This means the node expects the aggregated prompt field to contain an array and uses the first prompt
- **Key expressions or variables used:**
  - `={{ $('Chat Trigger').item.json.chatInput }}`
  - `={{ $json.Prompt[0] }}`
- **Input and output connections:**
  - Main input from `Aggregate Fields`
  - Receives language model from `GPT-4o-mini LLM`
  - Receives memory from `Teacher Session Memory`
  - Receives retrieval tool from `Book Knowledge Base`
  - Main output to `Log Q&A to Sheets`
- **Version-specific requirements:**
  - Uses node type version `1.9`
- **Edge cases / failures:**
  - If `Prompt[0]` is undefined, the agent may run with an empty or broken system prompt
  - If the LLM cannot call the retrieval tool correctly, answers may become generic
  - Long prompts plus memory plus retrieved context may hit token limits
  - If the trigger data is unavailable in expression context, the text field may error
- **Sub-workflow reference:** None

#### GPT-4o-mini LLM

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; OpenAI chat model used by the teacher agent.
- **Configuration choices:**
  - Model explicitly set to `gpt-4o-mini`
  - Default options otherwise
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `AI Teacher Agent` through `ai_languageModel`
- **Version-specific requirements:**
  - Uses node type version `1.2`
  - Requires OpenAI credentials and availability of `gpt-4o-mini`
- **Edge cases / failures:**
  - Invalid API key, insufficient quota, or model access restrictions
  - Latency spikes or provider-side timeouts
  - Model behavior may vary over time
- **Sub-workflow reference:** None

#### Teacher Session Memory

- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; rolling conversation memory for the tutoring agent.
- **Configuration choices:**
  - Custom session key mode
  - Session key:
    - `={{ $('Chat Trigger').item.json.sessionId }}`
  - Context window length: `10`
  - This stores the last 10 conversational turns/messages as configured by the memory implementation
- **Key expressions or variables used:**
  - `={{ $('Chat Trigger').item.json.sessionId }}`
- **Input and output connections:**
  - Connected to `AI Teacher Agent` through `ai_memory`
- **Version-specific requirements:**
  - Uses node type version `1.3`
- **Edge cases / failures:**
  - If `sessionId` is missing, memory continuity breaks
  - Short context window may omit important prior information
  - In-memory behavior may reset on workflow restarts depending on deployment/storage setup
- **Sub-workflow reference:** None

#### Book Knowledge Base

- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`; retrieval tool exposing Pinecone vector search to the teacher agent.
- **Configuration choices:**
  - Mode: `retrieve-as-tool`
  - `topK`: `3`
  - Tool name: `Pinecone_Vector_Store_Book`
  - Tool description: retrieve relevant educational content from the book vector store
  - Pinecone index: `rag-vector-db-book-quiz`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Receives embeddings from `OpenAI Embeddings`
  - Connected to `AI Teacher Agent` through `ai_tool`
- **Version-specific requirements:**
  - Uses node type version `1.1`
  - Requires Pinecone credentials and an existing index populated with documents
- **Edge cases / failures:**
  - Empty or poorly indexed vector database reduces response quality
  - Wrong dimensionality between embeddings and Pinecone index can break retrieval
  - Retrieval may surface irrelevant chunks if embeddings or chunking are weak
  - Provider auth or network errors
- **Sub-workflow reference:** None

#### OpenAI Embeddings

- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`; embedding model used by the Pinecone retrieval node for similarity search.
- **Configuration choices:**
  - Default options
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Book Knowledge Base` through `ai_embedding`
- **Version-specific requirements:**
  - Uses node type version `1.2`
  - Requires OpenAI credentials
- **Edge cases / failures:**
  - Auth issues or quota exhaustion
  - If embedding model configuration changes incompatibly with the Pinecone index, retrieval can fail
- **Sub-workflow reference:** None

---

## 2.4 Block: Logging & Response Return

### Overview

This block records the conversation in Google Sheets and prepares the chatbot response in the final output format. It preserves session traceability by storing the session ID, user message, and generated reply.

### Nodes Involved

- Log Q&A to Sheets
- Format Response Output

### Node Details

#### Log Q&A to Sheets

- **Type and role:** `n8n-nodes-base.googleSheets`; appends chat records to the preservation sheet.
- **Configuration choices:**
  - Operation: `append`
  - Document ID placeholder: `YOUR_GOOGLE_SHEET_ID`
  - Sheet placeholder: `YOUR_PRESERVATION_SHEET_GID`
  - Columns mapped explicitly:
    - `Session` = `={{ $('Chat Trigger').item.json.sessionId }}`
    - `User Msg` = `={{ $('Chat Trigger').item.json.chatInput }}`
    - `AI Response` = `={{ $json.output }}`
  - Type conversion options disabled
- **Key expressions or variables used:**
  - `={{ $('Chat Trigger').item.json.sessionId }}`
  - `={{ $('Chat Trigger').item.json.chatInput }}`
  - `={{ $json.output }}`
- **Input and output connections:**
  - Main input from `AI Teacher Agent`
  - Main output to `Format Response Output`
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.5`
- **Edge cases / failures:**
  - Placeholder IDs must be replaced
  - Sheet headers must match expected column names
  - If the teacher agent output field is not named `output`, log entry may be blank or fail
  - Append permissions and Google API quota can block execution
- **Sub-workflow reference:** None

#### Format Response Output

- **Type and role:** `n8n-nodes-base.set`; normalizes the final response into a simple JSON object for the chat UI.
- **Configuration choices:**
  - Raw JSON mode enabled
  - Output JSON:
    - `{"output": <AI Response as JSON-stringified value>}`
  - Specifically references `AI Response` from the prior Sheets append output
- **Key expressions or variables used:**
  - `={{ JSON.stringify($json["AI Response"]) }}`
- **Input and output connections:**
  - Input from `Log Q&A to Sheets`
  - No downstream node; terminal output
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases / failures:**
  - This node assumes the incoming item still contains `AI Response`
  - Some Google Sheets append operations return metadata rather than the original mapped row; if so, the expression may resolve to undefined
  - JSON stringification may wrap plain text in quotes, which may or may not be desired by the consuming chat frontend
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | n8n-nodes-base.stickyNote | Global documentation note describing purpose, setup, and customization |  |  | ### AI Tutor – Nathan (Interactive Learning Chatbot)<br>This workflow powers **Nathan**, an AI-driven educational tutor chatbot that guides users through personalised, topic-based learning sessions using Retrieval-Augmented Generation (RAG).<br>How it works: user sends a message, DeepSeek classifies intent, matching system prompt is fetched from Google Sheets, GPT-4o-mini answers with Pinecone retrieval, and each exchange is logged to Google Sheets.<br>Setup: add DeepSeek, OpenAI, Google Sheets, and Pinecone credentials; replace `YOUR_GOOGLE_SHEET_ID`; ensure `pmt` and `Preservation` tabs exist.<br>Customisation: swap classifier LLM, extend prompt library, adjust Pinecone `topK`. |
| Section – Receive & Classify Intent | n8n-nodes-base.stickyNote | Section note for intake and classification block |  |  | ## 1. Receive & Classify Intent<br>The chat trigger receives the user message. **DeepSeek** classifies it into one of five intents — *greeting, topic, answer, question, or random* — using a sliding-window memory for conversational context. |
| Section – Fetch & Merge Prompt | n8n-nodes-base.stickyNote | Section note for prompt lookup and merge block |  |  | ## 2. Fetch & Merge System Prompt<br>The classified intent looks up the matching system prompt from the Google Sheets prompt library. The prompt and the original chat input are then merged and aggregated before being forwarded to the AI Teacher. |
| Section – AI Teacher Agent | n8n-nodes-base.stickyNote | Section note for tutoring agent and RAG block |  |  | ## 3. AI Teacher Agent (RAG)<br>**GPT-4o-mini** generates a pedagogically appropriate response using the fetched system prompt, full session memory, and relevant content retrieved from the Pinecone book vector store. |
| Section – Log & Return Response | n8n-nodes-base.stickyNote | Section note for logging and response formatting block |  |  | ## 4. Log & Return Response<br>Appends the session ID, user message, and AI response to the **Preservation** sheet in Google Sheets. The final output is then formatted and returned to the chat interface. |
| DeepSeek LLM | @n8n/n8n-nodes-langchain.lmChatDeepSeek | LLM backend for intent classification |  | Intent Classifier | ## 1. Receive & Classify Intent<br>The chat trigger receives the user message. **DeepSeek** classifies it into one of five intents — *greeting, topic, answer, question, or random* — using a sliding-window memory for conversational context. |
| Intent Classifier | @n8n/n8n-nodes-langchain.agent | Classifies incoming user text into one of five intents | Chat Trigger; DeepSeek LLM; Classifier Memory | Fetch System Prompt | ## 1. Receive & Classify Intent<br>The chat trigger receives the user message. **DeepSeek** classifies it into one of five intents — *greeting, topic, answer, question, or random* — using a sliding-window memory for conversational context. |
| OpenAI Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embedding model for Pinecone retrieval |  | Book Knowledge Base | ## 3. AI Teacher Agent (RAG)<br>**GPT-4o-mini** generates a pedagogically appropriate response using the fetched system prompt, full session memory, and relevant content retrieved from the Pinecone book vector store. |
| GPT-4o-mini LLM | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main LLM used by the teaching agent |  | AI Teacher Agent | ## 3. AI Teacher Agent (RAG)<br>**GPT-4o-mini** generates a pedagogically appropriate response using the fetched system prompt, full session memory, and relevant content retrieved from the Pinecone book vector store. |
| Teacher Session Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Session-scoped rolling memory for the teaching agent |  | AI Teacher Agent | ## 3. AI Teacher Agent (RAG)<br>**GPT-4o-mini** generates a pedagogically appropriate response using the fetched system prompt, full session memory, and relevant content retrieved from the Pinecone book vector store. |
| Format Response Output | n8n-nodes-base.set | Formats final chat response JSON | Log Q&A to Sheets |  | ## 4. Log & Return Response<br>Appends the session ID, user message, and AI response to the **Preservation** sheet in Google Sheets. The final output is then formatted and returned to the chat interface. |
| AI Teacher Agent | @n8n/n8n-nodes-langchain.agent | Generates the tutor response using prompt, memory, and retrieval tool | Aggregate Fields; GPT-4o-mini LLM; Teacher Session Memory; Book Knowledge Base | Log Q&A to Sheets | ## 3. AI Teacher Agent (RAG)<br>**GPT-4o-mini** generates a pedagogically appropriate response using the fetched system prompt, full session memory, and relevant content retrieved from the Pinecone book vector store. |
| Chat Trigger | @n8n/n8n-nodes-langchain.chatTrigger | Public chat entry point and session source |  | Merge Input and Prompt; Intent Classifier | ## 1. Receive & Classify Intent<br>The chat trigger receives the user message. **DeepSeek** classifies it into one of five intents — *greeting, topic, answer, question, or random* — using a sliding-window memory for conversational context. |
| Aggregate Fields | n8n-nodes-base.aggregate | Aggregates prompt and input fields for the teaching agent | Merge Input and Prompt | AI Teacher Agent | ## 2. Fetch & Merge System Prompt<br>The classified intent looks up the matching system prompt from the Google Sheets prompt library. The prompt and the original chat input are then merged and aggregated before being forwarded to the AI Teacher. |
| Merge Input and Prompt | n8n-nodes-base.merge | Combines original chat payload with prompt lookup result | Chat Trigger; Fetch System Prompt | Aggregate Fields | ## 2. Fetch & Merge System Prompt<br>The classified intent looks up the matching system prompt from the Google Sheets prompt library. The prompt and the original chat input are then merged and aggregated before being forwarded to the AI Teacher. |
| Classifier Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memory context for intent classification |  | Intent Classifier | ## 1. Receive & Classify Intent<br>The chat trigger receives the user message. **DeepSeek** classifies it into one of five intents — *greeting, topic, answer, question, or random* — using a sliding-window memory for conversational context. |
| Fetch System Prompt | n8n-nodes-base.googleSheets | Retrieves prompt row matching classified intent | Intent Classifier | Merge Input and Prompt | ## 2. Fetch & Merge System Prompt<br>The classified intent looks up the matching system prompt from the Google Sheets prompt library. The prompt and the original chat input are then merged and aggregated before being forwarded to the AI Teacher. |
| Log Q&A to Sheets | n8n-nodes-base.googleSheets | Appends session, user message, and AI response to preservation log | AI Teacher Agent | Format Response Output | ## 4. Log & Return Response<br>Appends the session ID, user message, and AI response to the **Preservation** sheet in Google Sheets. The final output is then formatted and returned to the chat interface. |
| Book Knowledge Base | @n8n/n8n-nodes-langchain.vectorStorePinecone | Pinecone retrieval tool exposed to the teacher agent | OpenAI Embeddings | AI Teacher Agent | ## 3. AI Teacher Agent (RAG)<br>**GPT-4o-mini** generates a pedagogically appropriate response using the fetched system prompt, full session memory, and relevant content retrieved from the Pinecone book vector store. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Chat Trigger node**
   - Type: `Chat Trigger`
   - Enable **Public** mode
   - Set the initial message to:
     - `Hi there! 👋`
     - `My name is Nathan. Please share the topic you want to learn today!`
   - Keep note that this node will provide:
     - `chatInput`
     - `sessionId`

3. **Add an Intent Classifier agent**
   - Type: `AI Agent`
   - Set **Prompt Type** to define manually
   - Set **Text** to:
     - `{{ $json.chatInput }}`
   - Set the **System Message** to a strict classifier instruction that returns exactly one of:
     - `greeting`
     - `topic`
     - `answer`
     - `question`
     - `random`
   - Use the same logic as the source workflow:
     - greeting only if it is just a hello
     - topic if the user introduces a subject they want to learn
     - answer if replying to a previous question
     - question if asking something
     - random if off-topic or nonsense
     - include the instruction that if memory indicates an ongoing conversation, it is likely a question
   - Connect `Chat Trigger` to `Intent Classifier`

4. **Add a DeepSeek chat model**
   - Type: `DeepSeek Chat Model`
   - Leave default options unless you want custom temperature or model settings
   - Add **DeepSeek API credentials**
   - Connect this node to the `Intent Classifier` via the **language model** connector

5. **Add classifier memory**
   - Type: `Memory Buffer Window`
   - You can keep defaults to match the source workflow
   - Connect it to the `Intent Classifier` via the **memory** connector
   - If you want stronger session isolation than the source version, explicitly bind it to session ID

6. **Add a Google Sheets node for prompt lookup**
   - Type: `Google Sheets`
   - Configure Google Sheets credentials
   - Choose the spreadsheet that contains your prompt library
   - Replace the placeholder document ID with your real Google Sheet ID
   - Use the prompt library tab, named `pmt`
   - Configure it to look up rows where:
     - lookup column = `Output`
     - lookup value = `{{ $json.output }}`
   - This assumes the classifier node outputs its one-word classification in a field named `output`
   - Connect `Intent Classifier` to `Fetch System Prompt`

7. **Prepare the prompt sheet structure**
   - In Google Sheets, create a tab named `pmt`
   - Include at minimum:
     - a column named `Output` containing intent labels like `greeting`, `topic`, `answer`, `question`, `random`
     - a column named `Prompt` containing the corresponding system prompt text for the teacher agent
   - Ensure there is one clean match per intent

8. **Add a Merge node**
   - Type: `Merge`
   - Keep the default merge behavior unless you need a specific mode
   - Connect:
     - `Chat Trigger` to input 1 / left input
     - `Fetch System Prompt` to input 2 / right input
   - This node combines original user input with the retrieved prompt row

9. **Add an Aggregate node**
   - Type: `Aggregate`
   - Configure fields to aggregate:
     - `Prompt`
     - `chatInput`
   - Connect `Merge Input and Prompt` to `Aggregate Fields`
   - This reproduces the source behavior where the next agent reads `Prompt[0]`

10. **Add the AI Teacher agent**
    - Type: `AI Agent`
    - Set **Prompt Type** to define manually
    - Set **Text** to:
      - `{{ $('Chat Trigger').item.json.chatInput }}`
    - Set **System Message** to:
      - `{{ $json.Prompt[0] }}`
    - Connect `Aggregate Fields` to `AI Teacher Agent`

11. **Add the GPT-4o-mini model**
    - Type: `OpenAI Chat Model`
    - Choose model: `gpt-4o-mini`
    - Add **OpenAI credentials**
    - Connect it to `AI Teacher Agent` via the **language model** connector

12. **Add teacher session memory**
    - Type: `Memory Buffer Window`
    - Set session mode to custom key
    - Set session key to:
      - `{{ $('Chat Trigger').item.json.sessionId }}`
    - Set context window length to `10`
    - Connect it to `AI Teacher Agent` via the **memory** connector

13. **Add OpenAI embeddings**
    - Type: `OpenAI Embeddings`
    - Use the same or another valid OpenAI credential
    - Leave defaults unless your Pinecone index requires a specific embedding model
    - Connect later to the Pinecone vector store node

14. **Add a Pinecone vector store node**
    - Type: `Pinecone Vector Store`
    - Configure Pinecone credentials
    - Set mode to **retrieve as tool**
    - Select your Pinecone index:
      - source workflow uses `rag-vector-db-book-quiz`
    - Set `topK` to `3`
    - Set tool name to:
      - `Pinecone_Vector_Store_Book`
    - Set tool description to something like:
      - `Retrieve relevant educational content from the book vector store in Pinecone.`
    - Connect `OpenAI Embeddings` to this node via the **embedding** connector
    - Connect this Pinecone node to `AI Teacher Agent` via the **tool** connector

15. **Ensure your Pinecone index is prepared**
    - The index must already exist
    - It must contain educational/book chunks embedded with a compatible embedding model
    - Embedding dimension must match the index configuration

16. **Add a Google Sheets node for logging**
    - Type: `Google Sheets`
    - Operation: `Append`
    - Configure Google Sheets credentials
    - Select the same spreadsheet or another logging spreadsheet
    - Replace placeholders:
      - document ID = real sheet ID
      - sheet/tab = your `Preservation` sheet GID
    - Map columns:
      - `Session` = `{{ $('Chat Trigger').item.json.sessionId }}`
      - `User Msg` = `{{ $('Chat Trigger').item.json.chatInput }}`
      - `AI Response` = `{{ $json.output }}`
    - Connect `AI Teacher Agent` to `Log Q&A to Sheets`

17. **Prepare the preservation sheet**
    - Create a tab for logs, e.g. `Preservation`
    - Add these headers:
      - `Session`
      - `User Msg`
      - `AI Response`

18. **Add a Set node to format the final response**
    - Type: `Set`
    - Use raw JSON mode
    - Set output JSON to:
      - `{"output": {{ JSON.stringify($json["AI Response"]) }}}`
    - Connect `Log Q&A to Sheets` to `Format Response Output`

19. **Add optional sticky notes**
    - Add notes documenting:
      - overall purpose and setup
      - receive/classify section
      - fetch/merge section
      - teacher/RAG section
      - log/return section

20. **Credential checklist**
    - DeepSeek credential for the classifier model
    - OpenAI credential for:
      - GPT-4o-mini model
      - embeddings node
    - Google Sheets credential for:
      - prompt fetch node
      - logging node
    - Pinecone credential for the vector store

21. **Test the workflow**
    - Send a greeting such as `Hi`
      - expected classifier output: `greeting`
    - Send a topic request such as `I want to learn photosynthesis`
      - expected classifier output: `topic`
    - Ask a subject question
      - expected retrieval from Pinecone and a grounded teacher answer
    - Verify:
      - one prompt row is found
      - AI Teacher Agent gets a non-empty system prompt
      - log row is appended to the sheet
      - final chat output contains an `output` field

22. **Recommended hardening improvements**
    - Add a fallback if prompt lookup returns no rows
    - Normalize classifier output with lowercase trimming before Google Sheets lookup
    - Verify whether the Google Sheets append node returns `AI Response`; if not, format the output before logging or keep a copy of the teacher response in a dedicated node
    - Consider explicitly scoping classifier memory by `sessionId`
    - Consider adding moderation or rate limiting because the chat trigger is public

### Connection Order Summary

1. `Chat Trigger` → `Intent Classifier`
2. `DeepSeek LLM` → `Intent Classifier` as language model
3. `Classifier Memory` → `Intent Classifier` as memory
4. `Intent Classifier` → `Fetch System Prompt`
5. `Chat Trigger` → `Merge Input and Prompt`
6. `Fetch System Prompt` → `Merge Input and Prompt`
7. `Merge Input and Prompt` → `Aggregate Fields`
8. `Aggregate Fields` → `AI Teacher Agent`
9. `GPT-4o-mini LLM` → `AI Teacher Agent` as language model
10. `Teacher Session Memory` → `AI Teacher Agent` as memory
11. `OpenAI Embeddings` → `Book Knowledge Base` as embeddings
12. `Book Knowledge Base` → `AI Teacher Agent` as tool
13. `AI Teacher Agent` → `Log Q&A to Sheets`
14. `Log Q&A to Sheets` → `Format Response Output`

### Sub-workflow Setup

This workflow does **not** invoke any sub-workflow nodes and has **one entry point** only: `Chat Trigger`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Tutor – Nathan is an interactive educational chatbot using RAG, intent classification, GPT-4o-mini, Google Sheets prompt storage, and Google Sheets logging. | Workflow-wide |
| Required sheet structure: `pmt` tab for prompt library and `Preservation` tab for Q&A logs. | Google Sheets setup |
| Replace `YOUR_GOOGLE_SHEET_ID` and `YOUR_PRESERVATION_SHEET_GID` placeholders before running. | Google Sheets setup |
| You can swap DeepSeek for another classifier-capable LLM if needed. | Customization |
| You can extend the prompt library with additional intent categories, but then the classifier prompt and lookup design must be updated accordingly. | Customization |
| Pinecone `topK` is set to 3 and can be adjusted to change retrieval breadth. | RAG tuning |