Handle guest inquiries for multiple rentals with Pinecone Assistant and GPT-4.1

https://n8nworkflows.xyz/workflows/handle-guest-inquiries-for-multiple-rentals-with-pinecone-assistant-and-gpt-4-1-13467


# Handle guest inquiries for multiple rentals with Pinecone Assistant and GPT-4.1

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Handle guest inquiries for multiple rentals with Pinecone Assistant and GPT-4.1  
**Purpose:** Manage guest Q&A for multiple vacation rental properties by (1) uploading property documentation from Google Drive into separate Pinecone Assistants, and (2) routing guest chat questions to the correct Assistant using OpenAI (GPT‑4.1 mini) and LangChain Agents inside n8n.

**Primary use cases**
- A property manager maintains separate knowledge bases per property (Hillcrest, Birchwood, Lakeside).
- Guests ask questions (Wi‑Fi, appliance manuals, house rules, local recommendations).
- The workflow detects which property the question refers to and queries the appropriate Pinecone Assistant; if unclear, it asks follow-up questions.

### Logical blocks
**1.1 Property data ingestion (Drive → Pinecone Assistants)**  
Triggered when a file is added to a specific Google Drive folder; downloads the file and uploads it to the matching Pinecone Assistant.

**1.2 Chat intake + short-term memory**  
Receives a chat message, attaches short rolling conversation memory (3 messages), and passes it into an “AI Agent” classifier to infer property.

**1.3 Property detection + branching**  
Parses the classifier’s JSON. If a property was identified, route to a property-aware agent; otherwise return a “request more info” message.

**1.4 Property Q&A (Router Agent + Pinecone tools + OpenAI model)**  
A second agent can call one of three Pinecone Assistant *tool* nodes to fetch grounded answers for the selected property; also includes a rule to avoid tool calls if the guest asks to be contacted.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Upload property data to Pinecone Assistant (Step 1)
**Overview:** Watches three Google Drive folders (one per property). When a new file appears, it downloads the file and uploads it into the corresponding Pinecone Assistant using a stable `externalFileId` tied to the Drive file ID.

**Nodes involved:**
- Hillcrest file added
- Download new file
- Upload file to Hillcrest Assistant
- Birchwood file added
- Download new file1
- Upload file to Birchwood Assistant
- Lakeside file added
- Download new file2
- Upload file to Lakeside Assistant
- Sticky Note (Step 1 container)
- Sticky Note2 (global info panel; visually covers the whole template area)

#### Node: **Hillcrest file added**
- **Type / role:** Google Drive Trigger (`googleDriveTrigger`) — detects new files.
- **Config (interpreted):**
  - Event: `fileCreated`
  - Trigger: specific folder = Drive folder named/cached as **hillcrest** (folder ID `12nnk6Q6u1kS5_SMXRzpu-tYszD-ZbjlH`)
  - Polling: every minute
- **Outputs:** To **Download new file**
- **Failure/edge cases:**
  - OAuth token expiry / revoked consent
  - Polling delays; duplicate detection issues if Drive returns repeated events
  - Permissions: the OAuth principal must have access to the folder

#### Node: **Download new file**
- **Type / role:** Google Drive (`googleDrive`) — downloads the created file as binary.
- **Config:**
  - Operation: `download`
  - `fileId`: `{{$json.id}}` (from trigger event)
  - `fileName` option: `{{$json.name}}`
- **Inputs:** From **Hillcrest file added**
- **Outputs:** To **Upload file to Hillcrest Assistant**
- **Failure/edge cases:**
  - Large files can cause timeouts / memory pressure
  - Unsupported Google-native formats (Docs/Sheets) may download as exports depending on Drive behavior; verify file types

#### Node: **Upload file to Hillcrest Assistant**
- **Type / role:** Pinecone Assistant (`pineconeAssistant`) — uploads file into the Hillcrest assistant’s managed RAG store.
- **Config:**
  - Resource: `file`
  - Operation: `uploadFile`
  - Assistant target (inline JSON): name `n8n-vacation-rental-property-hillcrest`, host `https://prod-1-data.ke.pinecone.io`
  - `externalFileId`: `{{$json.id}}` (Google Drive file ID)
  - Additional fields: none
- **Inputs:** From **Download new file** (expects binary file content)
- **Failure/edge cases:**
  - Pinecone API auth/permission errors
  - Assistant name/host mismatch (wrong project/region)
  - If the node expects binary field naming conventions, ensure the download node outputs the binary in the default field used by the Pinecone node version

#### Node: **Birchwood file added**
- Same as Hillcrest trigger, but folder **birchwood** (folder ID `1GHDqU3-hDH2BoPpLZb6GCeH50pi9Fv1s`)
- **Outputs:** To **Download new file1**

#### Node: **Download new file1**
- Same as Download new file
- **Outputs:** To **Upload file to Birchwood Assistant**

#### Node: **Upload file to Birchwood Assistant**
- Same as Hillcrest upload, but assistant name `n8n-vacation-rental-property-birchwood`

#### Node: **Lakeside file added**
- Same trigger pattern, folder **lakeside** (folder ID `1cp7-JPn2b_sfs7ZKnXl6voC_kiopzu2T`)
- **Outputs:** To **Download new file2**

#### Node: **Download new file2**
- Same download pattern
- **Outputs:** To **Upload file to Lakeside Assistant**

#### Node: **Upload file to Lakeside Assistant**
- Same upload pattern, assistant name `n8n-vacation-rental-property-lakeside`

---

### Block 1.2 — Chat intake + memory (Step 2 start)
**Overview:** Receives chat messages, adds short conversational memory (3-message window), and runs an LLM-based classifier agent to infer which property the guest refers to.

**Nodes involved:**
- When chat message received
- Simple Memory
- OpenAI Chat Model
- AI Agent
- Chat Memory Manager
- Sticky Note1 (Step 2 container)
- Sticky Note2 (global info panel)

#### Node: **When chat message received**
- **Type / role:** Chat Trigger (`chatTrigger`) — entry point for chat-based interactions.
- **Config:**
  - Response mode: `lastNode` (the final node reached should return the response)
- **Outputs:** To **AI Agent**
- **Failure/edge cases:**
  - If downstream ends in a node not returning expected `output`, the chat response can be empty
  - Multi-branch flows: ensure one branch produces a final message consistently

#### Node: **Simple Memory**
- **Type / role:** Memory Buffer Window (`memoryBufferWindow`) — keeps a rolling window of chat context.
- **Config:**
  - `sessionIdType`: `customKey`
  - `sessionKey`: `{{$('When chat message received').item.json.sessionId}}`
  - Context window length: `3`
- **Connections:**
  - Output (ai_memory) → **Chat Memory Manager** and **AI Agent**
- **Failure/edge cases:**
  - If `sessionId` is missing from the trigger payload, memory will not associate correctly
  - Small window (3) may be insufficient if property is mentioned earlier; adjust length as needed

#### Node: **OpenAI Chat Model**
- **Type / role:** LangChain OpenAI Chat Model (`lmChatOpenAi`) — provides the model for the classifier agent.
- **Config:**
  - Model: `gpt-4.1-mini`
  - No extra options set
- **Connections:**
  - Output (ai_languageModel) → **AI Agent**
- **Failure/edge cases:**
  - OpenAI API key issues, quota/rate limits
  - Model name availability depends on your OpenAI account/region and n8n node version

#### Node: **AI Agent**
- **Type / role:** LangChain Agent (`agent`) — classifies the message into a property selection JSON or asks clarifying questions.
- **Config (key behavior):**
  - Max iterations: `3`
  - System message instructs:
    - Determine property among `"hillcrest" | "birchwood" | "lakeside"`
    - If known: “return only one json” containing `property: "<name>"`
    - Else: ask for more info in chat, friendly tone
- **Inputs:**
  - Main input from **When chat message received**
  - Memory from **Simple Memory**
  - Model from **OpenAI Chat Model**
- **Outputs:** To **Chat Memory Manager**
- **Failure/edge cases:**
  - **Critical:** The next block uses `JSON.parse($('AI Agent').item.json.output)`; if the agent returns non-JSON (e.g., clarifying question text), parsing will fail unless the “unknown property” branch still outputs valid JSON. As written, the system message explicitly asks it to ask for more info (non-JSON), which risks breaking the parse downstream.
  - Model may wrap JSON in Markdown fences unless strongly constrained; consider enforcing strict JSON mode or post-processing.

#### Node: **Chat Memory Manager**
- **Type / role:** Memory Manager (`memoryManager`) — consolidates messages and passes the conversation forward.
- **Config:** `groupMessages: true`
- **Inputs:** From **AI Agent** (+ memory stream from **Simple Memory**)
- **Outputs:** To **If property known**
- **Failure/edge cases:**
  - If message grouping changes structure, downstream nodes referencing `$json.messages[...]` must match the produced schema

---

### Block 1.3 — Property detection + branching
**Overview:** Attempts to parse the classifier output as JSON and checks whether `property` exists. If it does, proceeds to property Q&A; otherwise returns a follow-up request.

**Nodes involved:**
- If property known
- Request more info

#### Node: **If property known**
- **Type / role:** IF (`if`) — branch based on existence of the `property` field.
- **Config (interpreted):**
  - Condition: string “exists”
  - Left value expression:
    - `{{ JSON.parse($('AI Agent').item.json.output).property }}`
- **Connections:**
  - True → **Property Agent**
  - False → **Request more info**
- **Failure/edge cases:**
  - **Expression failure risk:** if `$('AI Agent').item.json.output` is not valid JSON, the IF node errors and the workflow fails (no chat response).
  - If the AI Agent returns JSON without `property`, false branch triggers; but the parse must still succeed.

#### Node: **Request more info**
- **Type / role:** Set (`set`) — maps agent clarifying text to a simple `output` field for chat response.
- **Config:**
  - Sets `output` = `{{$json.messages[0].ai}}`
- **Input:** From **If property known** (false path)
- **Outputs:** None (terminal)
- **Failure/edge cases:**
  - Depends on `Chat Memory Manager` output shape: `$json.messages[0].ai` must exist.
  - If the classifier agent didn’t produce an `ai` message in that position, the response will be empty.

---

### Block 1.4 — Property Q&A via Pinecone Assistant tools
**Overview:** Uses a second agent (“Property Agent”) that can call one of three Pinecone Assistant tools (one per property) to answer the guest’s questions. It uses an OpenAI model as the orchestrator LLM.

**Nodes involved:**
- Property Agent
- OpenAI Chat Model2
- Hillcrest Assistant (tool)
- Birchwood assistant (tool)
- Lakeside assistant (tool)

#### Node: **Property Agent**
- **Type / role:** LangChain Agent (`agent`) — tool-using agent to answer questions grounded in Pinecone.
- **Config:**
  - Prompt type: `define`
  - Text fed to agent: `{{$json.messages}}` (entire grouped message list)
  - System message:
    - “routes requests based on property name to the appropriate pinecone assistant”
    - If user requests to be contacted: **do not call tools**, return response that someone will reach out.
- **Inputs:**
  - Main input from **If property known** (true path)
  - Model from **OpenAI Chat Model2**
  - Tools from the 3 Pinecone Assistant Tool nodes
- **Outputs:** Not connected further (terminal)
- **Failure/edge cases:**
  - The agent is told to route “based on property name”, but the workflow does not explicitly pass the parsed `property` value into this agent as a dedicated field; it only passes `$json.messages`. If the messages don’t clearly contain the property name, routing may be unreliable. Consider injecting the parsed property into the agent input.
  - Since this is the “last node” in the true path, ensure it outputs in a format chat trigger can return (commonly `output` or a recognized response field depending on node behavior/version).

#### Node: **OpenAI Chat Model2**
- **Type / role:** OpenAI Chat Model (`lmChatOpenAi`) — LLM for Property Agent.
- **Config:** Model `gpt-4.1-mini`
- **Connections:** ai_languageModel → **Property Agent**
- **Failure/edge cases:** Same as OpenAI Chat Model node (auth/quota/model availability)

#### Node: **Hillcrest Assistant** (tool)
- **Type / role:** Pinecone Assistant Tool (`pineconeAssistantTool`) — exposes Hillcrest assistant as a callable tool to the agent.
- **Config:** assistant `n8n-vacation-rental-property-hillcrest`, host `https://prod-1-data.ke.pinecone.io`
- **Connections:** ai_tool → **Property Agent**
- **Failure/edge cases:** Pinecone auth; assistant missing; empty knowledge base if ingestion not completed

#### Node: **Birchwood assistant** (tool)
- Same as above for `n8n-vacation-rental-property-birchwood`

#### Node: **Lakeside assistant** (tool)
- Same as above for `n8n-vacation-rental-property-lakeside`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point | — | AI Agent | ## Step 2: Get answers about a specific property |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Classify/identify property from chat | When chat message received; Simple Memory; OpenAI Chat Model | Chat Memory Manager | ## Step 2: Get answers about a specific property |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for classifier agent | — | AI Agent | ## Step 2: Get answers about a specific property |
| Chat Memory Manager | @n8n/n8n-nodes-langchain.memoryManager | Groups messages & forwards | AI Agent; Simple Memory | If property known | ## Step 2: Get answers about a specific property |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Rolling chat memory (3 turns) | — | Chat Memory Manager; AI Agent | ## Step 2: Get answers about a specific property |
| If property known | n8n-nodes-base.if | Branch based on parsed property | Chat Memory Manager | Property Agent (true); Request more info (false) | ## Step 2: Get answers about a specific property |
| Request more info | n8n-nodes-base.set | Return clarifying question | If property known (false) | — | ## Step 2: Get answers about a specific property |
| Property Agent | @n8n/n8n-nodes-langchain.agent | Tool-using agent to answer property questions | If property known (true); OpenAI Chat Model2; Pinecone tools | — | ## Step 2: Get answers about a specific property |
| OpenAI Chat Model2 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Property Agent | — | Property Agent | ## Step 2: Get answers about a specific property |
| Hillcrest Assistant | @pinecone-database/n8n-nodes-pinecone-assistant.pineconeAssistantTool | Pinecone tool for Hillcrest KB | — | Property Agent | ## Step 2: Get answers about a specific property |
| Birchwood assistant | @pinecone-database/n8n-nodes-pinecone-assistant.pineconeAssistantTool | Pinecone tool for Birchwood KB | — | Property Agent | ## Step 2: Get answers about a specific property |
| Lakeside assistant | @pinecone-database/n8n-nodes-pinecone-assistant.pineconeAssistantTool | Pinecone tool for Lakeside KB | — | Property Agent | ## Step 2: Get answers about a specific property |
| Hillcrest file added | n8n-nodes-base.googleDriveTrigger | Detect new Hillcrest files | — | Download new file | ## Step 1: Upload property data to Pinecone Assistant |
| Download new file | n8n-nodes-base.googleDrive | Download new Hillcrest file | Hillcrest file added | Upload file to Hillcrest Assistant | ## Step 1: Upload property data to Pinecone Assistant |
| Upload file to Hillcrest Assistant | @pinecone-database/n8n-nodes-pinecone-assistant.pineconeAssistant | Upload Hillcrest docs to Pinecone Assistant | Download new file | — | ## Step 1: Upload property data to Pinecone Assistant |
| Birchwood file added | n8n-nodes-base.googleDriveTrigger | Detect new Birchwood files | — | Download new file1 | ## Step 1: Upload property data to Pinecone Assistant |
| Download new file1 | n8n-nodes-base.googleDrive | Download new Birchwood file | Birchwood file added | Upload file to Birchwood Assistant | ## Step 1: Upload property data to Pinecone Assistant |
| Upload file to Birchwood Assistant | @pinecone-database/n8n-nodes-pinecone-assistant.pineconeAssistant | Upload Birchwood docs to Pinecone Assistant | Download new file1 | — | ## Step 1: Upload property data to Pinecone Assistant |
| Lakeside file added | n8n-nodes-base.googleDriveTrigger | Detect new Lakeside files | — | Download new file2 | ## Step 1: Upload property data to Pinecone Assistant |
| Download new file2 | n8n-nodes-base.googleDrive | Download new Lakeside file | Lakeside file added | Upload file to Lakeside Assistant | ## Step 1: Upload property data to Pinecone Assistant |
| Upload file to Lakeside Assistant | @pinecone-database/n8n-nodes-pinecone-assistant.pineconeAssistant | Upload Lakeside docs to Pinecone Assistant | Download new file2 | — | ## Step 1: Upload property data to Pinecone Assistant |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block label | — | — | ## Step 1: Upload property data to Pinecone Assistant |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment block label | — | — | ## Step 2: Get answers about a specific property |
| Sticky Note2 | n8n-nodes-base.stickyNote | Global instructions/resources panel | — | — | ![Pinecone logo](https://www.pinecone.io/images/pinecone-logo-for-n8n-templates.png)<br>… (see Notes & Resources) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Pinecone Assistants (in Pinecone Console)**
   1. Create three Assistants:
      - `n8n-vacation-rental-property-lakeside`
      - `n8n-vacation-rental-property-birchwood`
      - `n8n-vacation-rental-property-hillcrest`
   2. Do not set special chat models/instructions there (the workflow uses OpenAI nodes in n8n).
   3. Note your Pinecone project/host (this workflow uses `https://prod-1-data.ke.pinecone.io`).

2) **Prepare Google Drive folders**
   1. Create three folders: `lakeside`, `birchwood`, `hillcrest`.
   2. Add markdown files (or your docs) into each folder; new file creation triggers ingestion.

3) **Create credentials in n8n**
   1. **Google Drive OAuth2** credential with access to the folders.
   2. **Pinecone Assistant API** credential with a valid API key for the project hosting the Assistants.
   3. **OpenAI API** credential with access to `gpt-4.1-mini` (or adjust model).

4) **Build Block 1: Drive ingestion (repeat per property)**
   - For **Hillcrest**:
   1. Add **Google Drive Trigger** node
      - Event: *File Created*
      - Trigger on: *Specific folder*
      - Folder: select `hillcrest`
      - Polling: every minute
   2. Add **Google Drive** node
      - Operation: *Download*
      - File ID: expression `{{$json.id}}`
      - Options → File Name: `{{$json.name}}`
   3. Add **Pinecone Assistant** node
      - Resource: *File*
      - Operation: *Upload File*
      - Assistant: set name `n8n-vacation-rental-property-hillcrest`
      - Host: `https://prod-1-data.ke.pinecone.io`
      - External File ID: `{{$json.id}}`
   4. Connect: Trigger → Download → Pinecone Upload
   - Repeat the same trio for **Birchwood** and **Lakeside**, changing the watched folder and assistant name.

5) **Build Block 2: Chat + property classifier**
   1. Add **When chat message received** (Chat Trigger)
      - Options → Response mode: `lastNode`
   2. Add **Simple Memory** (Memory Buffer Window)
      - Session ID type: Custom Key
      - Session key: `{{$('When chat message received').item.json.sessionId}}`
      - Context window length: `3`
   3. Add **OpenAI Chat Model**
      - Model: `gpt-4.1-mini`
      - Select your OpenAI credential
   4. Add **AI Agent** (LangChain Agent)
      - Options → Max iterations: `3`
      - System message: instruct classification into JSON with `property` ∈ {hillcrest,birchwood,lakeside}; otherwise ask for more info.
   5. Add **Chat Memory Manager**
      - Options: `groupMessages = true`
   6. Connect:
      - Chat Trigger → AI Agent (main)
      - Simple Memory → AI Agent (ai_memory)
      - Simple Memory → Chat Memory Manager (ai_memory)
      - OpenAI Chat Model → AI Agent (ai_languageModel)
      - AI Agent → Chat Memory Manager (main)

6) **Build Block 3: Branching**
   1. Add **If** node “If property known”
      - Condition: “exists”
      - Left value expression: `{{ JSON.parse($('AI Agent').item.json.output).property }}`
   2. Connect: Chat Memory Manager → If property known

7) **Build Block 4: Property Q&A agent with Pinecone tools**
   1. Add **OpenAI Chat Model** node (second one)
      - Model: `gpt-4.1-mini`
   2. Add **Pinecone Assistant Tool** nodes (3x)
      - Configure each with the assistant name + same host:
        - Hillcrest / Birchwood / Lakeside
   3. Add **Agent** node “Property Agent”
      - Prompt type: `define`
      - Text: `{{$json.messages}}`
      - System message: route to appropriate Pinecone assistant; if contact request, don’t call tools.
   4. Connect:
      - If property known (true) → Property Agent (main)
      - OpenAI Chat Model2 → Property Agent (ai_languageModel)
      - Each Pinecone Assistant Tool → Property Agent (ai_tool)

8) **False branch response**
   1. Add **Set** node “Request more info”
      - Set field `output` to: `{{$json.messages[0].ai}}`
   2. Connect: If property known (false) → Request more info

9) **Activate workflow**
   - Turn on the workflow.
   - Upload docs into each Drive folder to populate Pinecone.
   - Test via chat: ask an appliance question mentioning a property; also test an ambiguous question to see if it asks follow-ups.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Pinecone Assistant overview and benefits (managed RAG: chunking, embedding, vector search, orchestration, reranking). | https://docs.pinecone.io/guides/assistant/overview |
| Prerequisites: Pinecone account/API key; Google Drive API enabled + OAuth2 in n8n; OpenAI account/API key. | Pinecone console: https://app.pinecone.io/ • Google Drive OAuth guide: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/ • OpenAI keys: https://platform.openai.com/settings/organization/api-keys |
| Create three Pinecone Assistants with names: `n8n-vacation-rental-property-lakeside`, `...-birchwood`, `...-hillcrest`. | https://app.pinecone.io/organizations/-/projects/-/assistant |
| Example fictional data prompt (markdown files for house manuals, rules, wifi, local recommendations, appliance manuals). | Included in Sticky Note2 content |
| Optional sample files repository. | https://github.com/pinecone-io/n8n-templates/blob/main/vacation-rental-property-manager-assistants/fictional-data |
| Help channels: Pinecone Discord; GitHub issues. | Discord: https://discord.gg/tJ8V62S3sH • Issues: https://github.com/pinecone-io/n8n-templates/issues/new/choose |
| Template branding image (Pinecone logo). | https://www.pinecone.io/images/pinecone-logo-for-n8n-templates.png |