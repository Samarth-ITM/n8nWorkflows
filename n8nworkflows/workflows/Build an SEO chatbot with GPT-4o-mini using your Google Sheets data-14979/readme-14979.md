Build an SEO chatbot with GPT-4o-mini using your Google Sheets data

https://n8nworkflows.xyz/workflows/build-an-seo-chatbot-with-gpt-4o-mini-using-your-google-sheets-data-14979


# Build an SEO chatbot with GPT-4o-mini using your Google Sheets data

The user wants me to analyze an n8n workflow JSON and create a comprehensive, structured reference document. Let me carefully parse through the workflow JSON and all the provided description text to create the document.

Let me analyze the workflow structure:

**Nodes:**
1. "1. Chat Message Received" - @n8n/n8n-nodes-langchain.chatTrigger
2. "2. Set — Config Values" - n8n-nodes-base.set
3. "3. Google Sheets — Read Knowledge Base" - n8n-nodes-base.googleSheets
4. "4. Code — Build Knowledge Base Text" - n8n-nodes-base.code
5. "5. AI Agent — SEO Consultant" - @n8n/n8n-nodes-langchain.agent
6. "6. OpenAI — GPT-4o-mini Model" - @n8n/n8n-nodes-langchain.lmChatOpenAi
7. "7. Memory — Conversation Buffer" - @n8n/n8n-nodes-langchain.memoryBufferWindow
8. "8. Set — Prepare Log Fields" - n8n-nodes-base.set
9. "9. Google Sheets — Log Chat" - n8n-nodes-base.googleSheets
10. "10. Set — Return Answer to Chat" - n8n-nodes-base.set

Plus sticky notes:
- "Overview"
- "Section — Chat Input and Config"
- "Section — Knowledge Base Loading"
- "Section — AI Answer Generation"
- "Section — Logging and Chat Response"

**Connections:**
- Chat Message Received → Set Config Values
- Set Config Values → Google Sheets Read Knowledge Base
- Google Sheets Read Knowledge Base → Code Build Knowledge Base Text
- Code Build Knowledge Base Text → AI Agent SEO Consultant
- AI Agent SEO Consultant → Set Prepare Log Fields
- OpenAI GPT-4o-mini Model → AI Agent SEO Consultant (ai_languageModel)
- Memory Conversation Buffer → AI Agent SEO Consultant (ai_memory)
- Set Prepare Log Fields → Google Sheets Log Chat AND Set Return Answer to Chat (parallel)
- Google Sheets Read Knowledge Base operation: readOrSearch with documentId from expression

**Node 2 Config Values assignments:**
- knowledgeBaseSheetId: YOUR_KNOWLEDGE_BASE_SHEET_ID
- knowledgeBaseSheetName: SEO FAQ
- logSheetId: YOUR_CHAT_LOG_SHEET_ID
- logSheetName: Chat Log
- companyName: YOUR COMPANY NAME
- botPersona: "You are a professional SEO consultant at YOUR COMPANY NAME..."
- sessionId: {{ $now.toMillis().toString() }}

**Node 3 Google Sheets:**
- operation: readOrSearch
- documentId: mode=id, value= {{ $json.knowledgeBaseSheetId }}
- typeVersion: 4.5

**Node 4 Code:**
- Pulls all rows from input
- Gets config from "2. Set — Config Values"
- Gets userMessage from "1. Chat Message Received"
- If empty, returns fallback
- Builds numbered Q&A text block grouped by category
- Returns object with knowledgeBase, userMessage, sessionId, botPersona, companyName, logSheetId, logSheetName, timestamp

**Node 5 AI Agent:**
- text parameter is a multi-line expression combining botPersona, knowledgeBase, userMessage, and detailed instructions
- promptType: define
- Connected to OpenAI model and Memory

**Node 6 OpenAI:**
- model: gpt-4o-mini
- maxTokens: 400
- temperature: 0.3

**Node 7 Memory:**
- Buffer window memory, default parameters

**Node 8 Set Prepare Log Fields:**
- botAnswer: {{ $json.output || 'No response generated.' }}
- userQuestion: from Code node's userMessage
- sessionId: from Code node
- timestamp: from Code node
- logSheetId: from Code node
- logSheetName: from Code node

**Node 9 Google Sheets Log Chat:**
- operation: append
- documentId: {{ $json.logSheetId }}
- sheetName: {{ $json.logSheetName }}
- columns: Timestamp, Bot Answer, Session ID, User Question
- cellFormat: USER_ENTERED

**Node 10 Set Return Answer to Chat:**
- response: {{ $('8. Set — Prepare Log Fields').item.json.botAnswer }}

Now let me structure the document.

Logical blocks:
1. Chat Input and Config (nodes 1, 2)
2. Knowledge Base Loading (nodes 3, 4)
3. AI Answer Generation (nodes 5, 6, 7)
4. Logging and Chat Response (nodes 8, 9, 10)

Let me now build the document with all required sections.

For the sticky notes, I need to associate them with the relevant nodes:

- "Overview" - covers all nodes (it's a general overview sticky note)
- "Section — Chat Input and Config" - covers nodes 1 and 2
- "Section — Knowledge Base Loading" - covers nodes 3 and 4
- "Section — AI Answer Generation" - covers nodes 5, 6, 7
- "Section — Logging and Chat Response" - covers nodes 8, 9, 10

Plus the chat trigger node has its own notes in the node definition.

Let me build this out fully now.

Let me also check: The chat trigger node has a "notes" field with additional info about how to get the chat URL. I should include that in the Sticky Note column.

Actually, looking more carefully at the JSON, node "1. Chat Message Received" has a `notes` field (not a sticky note, but internal node notes). I should include that in the sticky note column or in the node details.

Let me compile the summary table and then the reproduction steps.

For the Summary Table sticky note column, I need to figure out which sticky notes cover which nodes based on position overlap. The positions of the sticky notes and nodes:

Nodes:
- 1. Chat Message Received: position [-1520, 192]
- 2. Set — Config Values: position [-1168, 192]
- 3. Google Sheets — Read Knowledge Base: position [-912, 192]
- 4. Code — Build Knowledge Base Text: position [-672, 192]
- 5. AI Agent — SEO Consultant: position [-320, 192]
- 6. OpenAI — GPT-4o-mini Model: position [-320, 400]
- 7. Memory — Conversation Buffer: position [-224, 544]
- 8. Set — Prepare Log Fields: position [144, 192]
- 9. Google Sheets — Log Chat: position [368, 32]
- 10. Set — Return Answer to Chat: position [368, 352]

Sticky notes:
- Overview: position [-2160, -224], width 508, height 1052 → covers roughly x: -2160 to -1652, y: -224 to 828. This is a general overview not covering specific nodes.
- Section — Chat Input and Config: position [-1600, 16], width 596, height 388 → x: -1600 to -1004, y: 16 to 404. Covers nodes 1 and 2.
- Section — Knowledge Base Loading: position [-976, 16], width 468, height 388 → x: -976 to -508, y: 16 to 404. Covers nodes 3 and 4.
- Section — AI Answer Generation: position [-400, -32], width 384, height 756 → x: -400 to -16, y: -32 to 724. Covers nodes 5, 6, 7.
- Section — Logging and Chat Response: position [96, -96], width 516, height 676 → x: 96 to 612, y: -96 to 580. Covers nodes 8, 9, 10.

Now the "Overview" sticky note is a big one at the far left - it's more of a general documentation note. Since it doesn't directly overlay the workflow nodes (it's to the left of them), I'll treat it as not directly covering any specific node, but it's a general workflow description.

Actually, per the instructions: "If a sticky note visually covers multiple nodes, duplicate its content in each affected row." The Overview sticky note is at x -2160 to -1652 which is far left, not directly over any of the workflow nodes (which start at x -1520). So it doesn't visually cover any node. I'll leave the sticky note column blank for nodes that don't have a covering sticky note, but I will include the "Section" sticky notes for the nodes they cover.

Wait, let me reconsider. The instructions say "Include any associated comment" in the sticky note column. The node "1. Chat Message Received" has its own `notes` field which is not a sticky note but a node-level note. I should include that as well, or is it only sticky notes?

Looking at the instructions: "The Sticky Note column must include any associated comment (no sticky note ID)." So I think it includes both sticky notes that cover the node AND the node's own notes.

Let me include the node notes for "1. Chat Message Received" and the section sticky notes for the relevant nodes.

Actually, I'll keep it to sticky notes only for the summary table, and include node notes in the detailed node analysis section.

Let me finalize the document structure.

Let me also think about edge cases and potential failures for each node.

Now, let me write the full document.### 1. Workflow Overview

This workflow creates a **branded SEO knowledge-base chatbot** that runs entirely inside n8n. A public n8n Chat URL receives visitor questions; the workflow reads the full Q&A knowledge base from a Google Sheet tab on every request, injects it into a GPT-4o-mini agent constrained to answer **only from the provided content**, returns the answer to the chat, and logs every exchange to a second Google Sheet tab for analytics. Conversation memory (buffer window) allows multi-turn follow-up questions within the same session.

The logic is organized into four sequential blocks:

| Block | Function |
|---|---|
| **1.1 Chat Input & Configuration** | Receives the user message via the n8n Chat Trigger and sets all runtime configuration (Sheet IDs, tab names, company name, bot persona, session ID). |
| **1.2 Knowledge Base Loading** | Reads all rows from the Google Sheets "SEO FAQ" tab and formats them into a structured text block that also carries the user's question and config values. |
| **1.3 AI Answer Generation** | Feeds the formatted knowledge base, bot persona, and user question into an AI Agent backed by GPT-4o-mini with conversation buffer memory; produces a strict, on-brand answer or an honest fallback message. |
| **1.4 Logging & Chat Response** | Extracts the bot answer and metadata into log fields, simultaneously appends a row to the Chat Log sheet and returns the answer to the chat interface. |

---

### 2. Block-by-Block Analysis

---

#### Block 1.1 — Chat Input & Configuration

**Overview**  
This block is the entry point. The Chat Trigger node exposes a public chat URL and forwards every incoming message. The Set node then centralises all runtime parameters (Sheet IDs, tab names, company name, bot persona) and generates a unique session ID per conversation.

**Nodes Involved**  
- 1. Chat Message Received  
- 2. Set — Config Values

---

**Node: 1. Chat Message Received**

| Attribute | Detail |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.chatTrigger` (v1.1) |
| Role | Creates a public n8n Chat interface; receives each user message as the workflow trigger. |
| Configuration | Webhook ID: `seo-chatbot-kb-001`. Options left at defaults (title, subtitle, inputPlaceholder can be customised). |
| Key Expressions / Variables | Output field `chatInput` contains the user's message text. |
| Input | None (trigger node). |
| Output | Main output → **2. Set — Config Values**. |
| Credentials | None required. |
| Edge Cases / Failure Types | • Workflow must be **Active** for the chat URL to respond. If inactive, visitors see no response. <br>• If multiple users hit the URL simultaneously, each triggers a separate execution — session IDs differentiate them. <br>• The webhook ID must be unique across the n8n instance; renaming or duplicating the workflow without changing it can cause conflicts. |
| Node-level Notes | CHAT TRIGGER — No credentials needed. This creates an n8n Chat interface at a public URL. HOW TO GET THE CHAT URL: 1. Activate the workflow 2. Click on this node 3. Copy the Chat URL shown 4. Share this URL with your team or embed it. Customize the chat UI in parameters: title → Chat window heading, subtitle → Description shown under title, inputPlaceholder → Hint text in the input box. The chat supports multi-turn conversation (user can ask follow-up questions naturally). |

---

**Node: 2. Set — Config Values**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.set` (v3.4) |
| Role | Stores all configuration values needed by downstream nodes and generates a unique session ID. |
| Configuration — Assignments | |
| `knowledgeBaseSheetId` | String — placeholder `YOUR_KNOWLEDGE_BASE_SHEET_ID` (must be replaced with actual Sheet ID). |
| `knowledgeBaseSheetName` | String — default `SEO FAQ` (must match the tab name exactly, case-sensitive). |
| `logSheetId` | String — placeholder `YOUR_CHAT_LOG_SHEET_ID` (can be the same Sheet ID or a different one). |
| `logSheetName` | String — default `Chat Log` (must match the log tab name exactly). |
| `companyName` | String — placeholder `YOUR COMPANY NAME` (used in fallback message). |
| `botPersona` | String — default: *"You are a professional SEO consultant at YOUR COMPANY NAME. You only answer questions about SEO, digital marketing, and website optimization."* (customise brand voice here). |
| `sessionId` | String — expression `{{ $now.toMillis().toString() }}` (generates a unique numeric ID per execution). |
| Input | From **1. Chat Message Received**. |
| Output | To **3. Google Sheets — Read Knowledge Base**. All fields are available downstream as `$json.fieldName`. |
| Edge Cases / Failure Types | • If Sheet IDs are not replaced, the Google Sheets nodes will fail with "Spreadsheet not found." <br>• If tab names have trailing spaces or different casing, the Sheets nodes will fail. <br>• The `sessionId` uses millisecond timestamps — extremely rapid simultaneous requests could theoretically collide, but in practice this is negligible. |

**Sticky Note covering this block**  
*Section — Chat Input and Config: "The Chat Trigger creates a public chat interface. Config stores both Sheet IDs, tab names, company name, bot persona, and a session ID generated per conversation."*

---

#### Block 1.2 — Knowledge Base Loading

**Overview**  
This block pulls every row from the "SEO FAQ" tab, then the Code node formats those rows into a structured, numbered Q&A text block. It also extracts the user's original message and carries forward all config values needed later (session ID, log Sheet info, etc.). If the sheet is empty, a safe fallback message is produced instead of crashing.

**Nodes Involved**  
- 3. Google Sheets — Read Knowledge Base  
- 4. Code — Build Knowledge Base Text

---

**Node: 3. Google Sheets — Read Knowledge Base**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Role | Reads all rows from the SEO FAQ knowledge base tab. |
| Configuration | Operation: `readOrSearch`. Document ID: expression `{{ $json.knowledgeBaseSheetId }}` (mode: id). Sheet name / range defaults to the tab specified in config (in this version the sheet name comes from the Set node via `knowledgeBaseSheetName` — note: the node configuration only shows `documentId` as an expression; the sheet/tab name must also be set, typically via `sheetName` parameter or defaults to the first tab). |
| Key Expressions | `documentId` = `{{ $json.knowledgeBaseSheetId }}` |
| Input | From **2. Set — Config Values**. |
| Output | Each row of the SEO FAQ tab is emitted as a separate item (one item per row). All rows flow to **4. Code — Build Knowledge Base Text**. |
| Credentials | Google Sheets OAuth2 — must be connected and authorised for the account that owns or can access the target spreadsheet. |
| Edge Cases / Failure Types | • OAuth token expiry → re-authenticate. <br>• Wrong Sheet ID → "Spreadsheet not found" error. <br>• Tab name mismatch → returns empty or wrong data. <br>• Sheet is empty → returns zero items; the Code node handles this gracefully. <br>• Permission denied → ensure the Google account has at least Viewer access. |

---

**Node: 4. Code — Build Knowledge Base Text**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.code` (v2) |
| Role | Formats all Q&A rows into a single structured text block and bundles it with the user message and config values for the AI Agent. |
| Key Logic (summarised) | 1. Collects all items from the Sheets read (`$input.all()`). <br>2. Retrieves config from node **2. Set — Config Values** and the user message from node **1. Chat Message Received** (`item.json.chatInput`). <br>3. If the sheet is empty, returns a fallback message: *"Knowledge base is empty. Please add Q&A rows to the SEO FAQ sheet."* <br>4. Otherwise iterates each row, extracting `Question`, `Answer`, `Category` (falls back to `question`, `answer`, `category` — case-insensitive columns). <br>5. Builds text: `Q{n} [Category]: Question\nA{n}: Answer\n\n` for each valid pair. <br>6. Returns a single item with fields: `knowledgeBase`, `userMessage`, `sessionId`, `botPersona`, `companyName`, `logSheetId`, `logSheetName`, `timestamp` (ISO format truncated to 16 chars). |
| Key Expressions | `$input.all()`, `$('2. Set — Config Values').item.json`, `$('1. Chat Message Received').item.json.chatInput` |
| Input | From **3. Google Sheets — Read Knowledge Base** (multiple items). |
| Output | Single item to **5. AI Agent — SEO Consultant**. |
| Edge Cases / Failure Types | • If `chatInput` is undefined/null → `userMessage` becomes empty string. <br>• If rows have unexpected column names → they won't be picked up (the code tries both capitalised and lowercase variants). <br>• If all rows have empty Question or Answer → returns *"No valid Q&A pairs found in knowledge base."* <br>• The timestamp is truncated to 16 characters (`YYYY-MM-DD HH:MM`) — seconds and timezone info are discarded. |

**Sticky Note covering this block**  
*Section — Knowledge Base Loading: "Reads all Q&A rows from the Google Sheets SEO FAQ tab. Formats them into a numbered Q&A text block. Also pulls the user message from the Chat Trigger for passing to the AI."*

---

#### Block 1.3 — AI Answer Generation

**Overview**  
The AI Agent receives the full knowledge base text, the bot persona, and the user's question. Instructed to answer **only** from the provided knowledge base, it either responds with a relevant answer or outputs a strict fallback message directing the user to company support. The OpenAI language model and a conversation buffer memory are attached as sub-nodes.

**Nodes Involved**  
- 5. AI Agent — SEO Consultant  
- 6. OpenAI — GPT-4o-mini Model  
- 7. Memory — Conversation Buffer

---

**Node: 5. AI Agent — SEO Consultant**

| Attribute | Detail |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.agent` (v1.7) |
| Role | Orchestrates the LLM call: receives the knowledge base and user question, applies the persona and strict instructions, and returns the answer. |
| Configuration | Prompt type: `define`. The `text` parameter is a multi-line expression that assembles: <br>• Bot persona (`{{ $json.botPersona }}`) <br>• Knowledge base block (`KNOWLEDGE BASE: {{ $json.knowledgeBase }}`) <br>• Current user question (`CURRENT QUESTION FROM USER: {{ $json.userMessage }}`) <br>• Six numbered instructions: (1) search KB carefully, (2) rephrase/expand if found, (3) exact fallback message if not found, (4) ≤ 150 words, (5) plain text only, (6) friendly and professional tone. |
| Fallback message template | *"I do not have information about this topic in my current knowledge base. Please contact {{ $json.companyName }} support directly for help with this."* |
| Connected Sub-nodes | `ai_languageModel` ← **6. OpenAI — GPT-4o-mini Model** <br>`ai_memory` ← **7. Memory — Conversation Buffer** |
| Input | From **4. Code — Build Knowledge Base Text**. |
| Output | Main output → **8. Set — Prepare Log Fields**. The agent's text response is available in `$json.output`. |
| Edge Cases / Failure Types | • If the OpenAI API key is invalid or credits are exhausted → API error. <br>• If the knowledge base text is very long, token limits could be approached; monitor total prompt tokens. <br>• If `output` is null/empty (unlikely but possible on API timeout), node 8 falls back to `'No response generated.'`. <br>• Hallucination risk is mitigated by the strict prompt, but very long KBs or ambiguous prompts could still cause off-topic answers; lowering the model temperature further (from 0.3 to 0.1) reduces this risk. |

---

**Node: 6. OpenAI — GPT-4o-mini Model**

| Attribute | Detail |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) |
| Role | Language model powering the AI Agent. |
| Configuration | Model: `gpt-4o-mini` (selected from list). Options: `maxTokens` = 400, `temperature` = 0.3. |
| Credentials | OpenAI API key — must be connected and have access to `gpt-4o-mini`. |
| Input | Connected to **5. AI Agent — SEO Consultant** via `ai_languageModel` link. |
| Output | Feeds model responses back to the Agent node. |
| Edge Cases / Failure Types | • Rate limits on the OpenAI API → 429 errors. <br>• Insufficient credits → 402 / authentication failure. <br>• Model name change or deprecation → update `value` field. <br>• `maxTokens` = 400 caps output length; very detailed answers may be truncated. Increase to ~700 if longer answers are needed. |

---

**Node: 7. Memory — Conversation Buffer**

| Attribute | Detail |
|---|---|
| Type | `@n8n/n8n-nodes-langchain.memoryBufferWindow` (v1.3) |
| Role | Maintains conversation history for the current session so users can ask follow-up questions naturally. |
| Configuration | Default parameters (no customisation). Uses n8n's built-in buffer window memory which keeps a sliding window of recent exchanges per session. |
| Input | Connected to **5. AI Agent — SEO Consultant** via `ai_memory` link. |
| Output | Feeds stored context back to the Agent on each turn. |
| Edge Cases / Failure Types | • Memory is session-scoped — if the chat window is refreshed/closed, a new session starts and history is lost. <br>• Very long conversations could consume significant context tokens; the buffer window default limits this, but if customised to retain all history, token costs may increase. <br>• No persistent storage beyond the n8n execution context — memory is not written to Sheets. |

**Sticky Note covering this block**  
*Section — AI Answer Generation: "GPT-4o-mini answers strictly from the injected knowledge base. If the question is not covered, it returns an honest fallback message. Conversation memory keeps context for follow-up questions in the same session."*

---

#### Block 1.4 — Logging & Chat Response

**Overview**  
After the AI Agent produces an answer, this block splits into two parallel paths: (a) append a row to the Chat Log sheet with session ID, timestamp, question, and answer; (b) return the answer to the chat interface. Both run simultaneously so the user sees the reply instantly while logging proceeds in the background.

**Nodes Involved**  
- 8. Set — Prepare Log Fields  
- 9. Google Sheets — Log Chat  
- 10. Set — Return Answer to Chat

---

**Node: 8. Set — Prepare Log Fields**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.set` (v3.4) |
| Role | Normalises the agent's output and gathers all metadata needed for logging and for the chat response. |
| Configuration — Assignments | |
| `botAnswer` | `{{ $json.output || 'No response generated.' }}` — captures the agent's text output; falls back to a default message if `output` is empty. |
| `userQuestion` | `{{ $('4. Code — Build Knowledge Base Text').item.json.userMessage }}` — retrieves the original user message from the Code node. |
| `sessionId` | `{{ $('4. Code — Build Knowledge Base Text').item.json.sessionId }}` — retrieves the session ID. |
| `timestamp` | `{{ $('4. Code — Build Knowledge Base Text').item.json.timestamp }}` — retrieves the pre-formatted timestamp. |
| `logSheetId` | `{{ $('4. Code — Build Knowledge Base Text').item.json.logSheetId }}` — retrieves the log Sheet ID. |
| `logSheetName` | `{{ $('4. Code — Build Knowledge Base Text').item.json.logSheetName }}` — retrieves the log tab name. |
| Input | From **5. AI Agent — SEO Consultant**. |
| Output | Two parallel outputs: <br>• To **9. Google Sheets — Log Chat** (index 0) <br>• To **10. Set — Return Answer to Chat** (index 0) |
| Edge Cases / Failure Types | • If the Code node didn't produce the expected fields (e.g., session ID missing), expressions will evaluate to `undefined` which could cause the Sheets append to fail. <br>• Cross-node references (`$('4. Code — Build Knowledge Base Text')`) depend on the Code node having run successfully; if it errored, this node will also fail. |

---

**Node: 9. Google Sheets — Log Chat**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Role | Appends a new row to the Chat Log tab for every Q&A exchange. |
| Configuration | Operation: `append`. <br>Document ID: `{{ $json.logSheetId }}` (mode: id). <br>Sheet name: `{{ $json.logSheetName }}` (mode: name). <br>Columns mapping (defineBelow): <br>• `Timestamp` = `{{ $json.timestamp }}` <br>• `Bot Answer` = `{{ $json.botAnswer }}` <br>• `Session ID` = `{{ $json.sessionId }}` <br>• `User Question` = `{{ $json.userQuestion }}` <br>Options: `cellFormat` = `USER_ENTERED`. |
| Credentials | Google Sheets OAuth2 — must be connected separately here (can reuse the same credential from node 3). The account must have edit access to the log spreadsheet. |
| Input | From **8. Set — Prepare Log Fields**. |
| Output | None (terminal node for this branch). |
| Edge Cases / Failure Types | • If the target sheet doesn't have the exact column headers (`Session ID`, `Timestamp`, `User Question`, `Bot Answer`) in row 1, the append operation may create a new row with misaligned data. <br>• If `logSheetId` is wrong or the tab name doesn't exist → "Spreadsheet not found" or "Sheet not found" error. <br>• OAuth token must be valid; expired tokens require re-authentication. <br>• Very long bot answers could exceed Google Sheets cell limits (50,000 characters). |

---

**Node: 10. Set — Return Answer to Chat**

| Attribute | Detail |
|---|---|
| Type | `n8n-nodes-base.set` (v3.4) |
| Role | Formats the final response that the Chat Trigger sends back to the user's browser. |
| Configuration — Assignments | |
| `response` | `{{ $('8. Set — Prepare Log Fields').item.json.botAnswer }}` — pulls the bot answer from the Prepare Log Fields node. |
| Input | From **8. Set — Prepare Log Fields**. |
| Output | Returned to the Chat Trigger interface (the Chat Trigger reads the `response` field and displays it). |
| Edge Cases / Failure Types | • If node 8 didn't produce `botAnswer`, the expression resolves to `undefined` and the chat may display "undefined" or an empty message. <br>• This node runs in parallel with the logging node; if logging is slow, the user still receives the answer immediately. |

**Sticky Note covering this block**  
*Section — Logging and Chat Response: "Every Q&A exchange is logged to the Chat Log sheet for analytics and KB improvement. The bot answer is also returned to the chat interface simultaneously."*

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 1. Chat Message Received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point; creates public chat URL and receives user messages | None (trigger) | 2. Set — Config Values | Section — Chat Input and Config: "The Chat Trigger creates a public chat interface. Config stores both Sheet IDs, tab names, company name, bot persona, and a session ID generated per conversation." — Also has node-level note: "CHAT TRIGGER — No credentials needed. This creates an n8n Chat interface at a public URL. HOW TO GET THE CHAT URL: 1. Activate the workflow 2. Click on this node 3. Copy the Chat URL shown 4. Share this URL with your team or embed it. Customize the chat UI in parameters: title → Chat window heading, subtitle → Description shown under title, inputPlaceholder → Hint text in the input box. The chat supports multi-turn conversation (user can ask follow-up questions naturally)." |
| 2. Set — Config Values | n8n-nodes-base.set | Stores configuration (Sheet IDs, tab names, company name, bot persona, session ID) | 1. Chat Message Received | 3. Google Sheets — Read Knowledge Base | Section — Chat Input and Config: "The Chat Trigger creates a public chat interface. Config stores both Sheet IDs, tab names, company name, bot persona, and a session ID generated per conversation." |
| 3. Google Sheets — Read Knowledge Base | n8n-nodes-base.googleSheets | Reads all Q&A rows from the SEO FAQ tab | 2. Set — Config Values | 4. Code — Build Knowledge Base Text | Section — Knowledge Base Loading: "Reads all Q&A rows from the Google Sheets SEO FAQ tab. Formats them into a numbered Q&A text block. Also pulls the user message from the Chat Trigger for passing to the AI." |
| 4. Code — Build Knowledge Base Text | n8n-nodes-base.code | Formats sheet rows into a structured text block; extracts user message and config values | 3. Google Sheets — Read Knowledge Base | 5. AI Agent — SEO Consultant | Section — Knowledge Base Loading: "Reads all Q&A rows from the Google Sheets SEO FAQ tab. Formats them into a numbered Q&A text block. Also pulls the user message from the Chat Trigger for passing to the AI." |
| 5. AI Agent — SEO Consultant | @n8n/n8n-nodes-langchain.agent | Orchestrates GPT-4o-mini to answer strictly from the knowledge base | 4. Code — Build Knowledge Base Text; 6. OpenAI — GPT-4o-mini Model (ai_languageModel); 7. Memory — Conversation Buffer (ai_memory) | 8. Set — Prepare Log Fields | Section — AI Answer Generation: "GPT-4o-mini answers strictly from the injected knowledge base. If the question is not covered, it returns an honest fallback message. Conversation memory keeps context for follow-up questions in the same session." |
| 6. OpenAI — GPT-4o-mini Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the LLM (gpt-4o-mini) to the AI Agent | Connected to 5. AI Agent — SEO Consultant | Feeds into 5. AI Agent — SEO Consultant (ai_languageModel) | Section — AI Answer Generation: "GPT-4o-mini answers strictly from the injected knowledge base. If the question is not covered, it returns an honest fallback message. Conversation memory keeps context for follow-up questions in the same session." |
| 7. Memory — Conversation Buffer | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation history per session for multi-turn context | Connected to 5. AI Agent — SEO Consultant | Feeds into 5. AI Agent — SEO Consultant (ai_memory) | Section — AI Answer Generation: "GPT-4o-mini answers strictly from the injected knowledge base. If the question is not covered, it returns an honest fallback message. Conversation memory keeps context for follow-up questions in the same session." |
| 8. Set — Prepare Log Fields | n8n-nodes-base.set | Normalises agent output and collects metadata for logging and response | 5. AI Agent — SEO Consultant | 9. Google Sheets — Log Chat; 10. Set — Return Answer to Chat | Section — Logging and Chat Response: "Every Q&A exchange is logged to the Chat Log sheet for analytics and KB improvement. The bot answer is also returned to the chat interface simultaneously." |
| 9. Google Sheets — Log Chat | n8n-nodes-base.googleSheets | Appends the Q&A exchange to the Chat Log tab | 8. Set — Prepare Log Fields | None (terminal) | Section — Logging and Chat Response: "Every Q&A exchange is logged to the Chat Log sheet for analytics and KB improvement. The bot answer is also returned to the chat interface simultaneously." |
| 10. Set — Return Answer to Chat | n8n-nodes-base.set | Sends the bot answer back to the chat interface | 8. Set — Prepare Log Fields | Chat Trigger (response displayed to user) | Section — Logging and Chat Response: "Every Q&A exchange is logged to the Chat Log sheet for analytics and KB improvement. The bot answer is also returned to the chat interface simultaneously." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance. Name it *"Build an SEO chatbot with GPT-4o-mini using your Google Sheets data"*.

2. **Add a Chat Trigger node**  
   - Type: `Chat Trigger` (n8n-nodes-langchain.chatTrigger, v1.1).  
   - Leave all options at default. Optionally customise title, subtitle, and inputPlaceholder under the Options section to brand your chat window.  
   - Note the Webhook ID (e.g., `seo-chatbot-kb-001`); if you duplicate the workflow, change it to avoid conflicts.  
   - This node requires no credentials.

3. **Add a Set node — "2. Set — Config Values"**  
   - Type: `Set` (n8n-nodes-base.set, v3.4).  
   - Add the following string assignments:  
     - `knowledgeBaseSheetId` → `YOUR_KNOWLEDGE_BASE_SHEET_ID` (replace later)  
     - `knowledgeBaseSheetName` → `SEO FAQ` (or your custom tab name)  
     - `logSheetId` → `YOUR_CHAT_LOG_SHEET_ID` (replace later)  
     - `logSheetName` → `Chat Log` (or your custom tab name)  
     - `companyName` → `YOUR COMPANY NAME` (replace later)  
     - `botPersona` → `You are a professional SEO consultant at YOUR COMPANY NAME. You only answer questions about SEO, digital marketing, and website optimization.` (customise as needed)  
     - `sessionId` → Expression: `{{ $now.toMillis().toString() }}`  
   - Connect: **Chat Trigger → this Set node**.

4. **Add a Google Sheets node — "3. Google Sheets — Read Knowledge Base"**  
   - Type: `Google Sheets` (n8n-nodes-base.googleSheets, v4.5).  
   - Operation: `readOrSearch`.  
   - Document ID: switch to expression mode → `{{ $json.knowledgeBaseSheetId }}`.  
   - Sheet Name / Range: set to `{{ $json.knowledgeBaseSheetName }}` or type `SEO FAQ` if using a fixed name.  
   - Connect a **Google Sheets OAuth2** credential (sign in with the Google account that owns or can access the knowledge base sheet).  
   - Connect: **2. Set — Config Values → this node**.

5. **Add a Code node — "4. Code — Build Knowledge Base Text"**  
   - Type: `Code` (n8n-nodes-base.code, v2).  
   - Paste the following JavaScript logic (summarised):  
     - Retrieve all input items (`$input.all()`).  
     - Get config from `$('2. Set — Config Values').item.json`.  
     - Get user message from `$('1. Chat Message Received').item.json.chatInput`.  
     - If no rows, return fallback: `"Knowledge base is empty. Please add Q&A rows to the SEO FAQ sheet."`  
     - Loop through each row, extract `Question`, `Answer`, `Category` (supporting lowercase variants).  
     - Build text: `Q{n} [Category]: Question\nA{n}: Answer\n\n` for each valid pair.  
     - Return single item with fields: `knowledgeBase`, `userMessage`, `sessionId`, `botPersona`, `companyName`, `logSheetId`, `logSheetName`, `timestamp` (ISO format truncated to 16 chars).  
   - Connect: **3. Google Sheets — Read Knowledge Base → this node**.

6. **Add an AI Agent node — "5. AI Agent — SEO Consultant"**  
   - Type: `AI Agent` (@n8n/n8n-nodes-langchain.agent, v1.7).  
   - Prompt Type: `define`.  
   - Text field — compose as a single expression combining:  
     ```
     {{ $json.botPersona }}

     You have access to the following SEO knowledge base. Use ONLY this knowledge base to answer questions. Do not make up answers. Do not use outside knowledge.

     KNOWLEDGE BASE:
     {{ $json.knowledgeBase }}

     CURRENT QUESTION FROM USER:
     {{ $json.userMessage }}

     INSTRUCTIONS:
     1. Search the knowledge base carefully for the answer to this question.
     2. If you find a relevant answer, respond clearly and helpfully. You can rephrase and expand the answer in your own words.
     3. If the question is not covered in the knowledge base, say exactly: I do not have information about this topic in my current knowledge base. Please contact {{ $json.companyName }} support directly for help with this.
     4. Keep answers concise — under 150 words.
     5. Plain text only. No markdown. No bullet symbols. No asterisks.
     6. Be friendly and professional.
     ```  
   - Connect: **4. Code — Build Knowledge Base Text → this node** (main input).  
   - Sub-node connections will be added in steps 7 and 8.

7. **Add an OpenAI Model node — "6. OpenAI — GPT-4o-mini Model"**  
   - Type: `OpenAI Chat Model` (@n8n/n8n-nodes-langchain.lmChatOpenAi, v1.2).  
   - Model: select `gpt-4o-mini` from the list.  
   - Options: set `maxTokens` to 400 and `temperature` to 0.3.  
   - Connect an **OpenAI API key** credential.  
   - Connect: Link the **ai_languageModel** output of this node to the **ai_languageModel** input of **5. AI Agent — SEO Consultant**.

8. **Add a Memory node — "7. Memory — Conversation Buffer"**  
   - Type: `Buffer Window Memory` (@n8n/n8n-nodes-langchain.memoryBufferWindow, v1.3).  
   - Leave all parameters at default (uses standard buffer window).  
   - Connect: Link the **ai_memory** output of this node to the **ai_memory** input of **5. AI Agent — SEO Consultant**.

9. **Add a Set node — "8. Set — Prepare Log Fields"**  
   - Type: `Set` (n8n-nodes-base.set, v3.4).  
   - Assignments:  
     - `botAnswer` → `{{ $json.output || 'No response generated.' }}`  
     - `userQuestion` → `{{ $('4. Code — Build Knowledge Base Text').item.json.userMessage }}`  
     - `sessionId` → `{{ $('4. Code — Build Knowledge Base Text').item.json.sessionId }}`  
     - `timestamp` → `{{ $('4. Code — Build Knowledge Base Text').item.json.timestamp }}`  
     - `logSheetId` → `{{ $('4. Code — Build Knowledge Base Text').item.json.logSheetId }}`  
     - `logSheetName` → `{{ $('4. Code — Build Knowledge Base Text').item.json.logSheetName }}`  
   - Connect: **5. AI Agent — SEO Consultant → this node**.  
   - This node has **two outputs** wired to the next two nodes in parallel.

10. **Add a Google Sheets node — "9. Google Sheets — Log Chat"**  
    - Type: `Google Sheets` (n8n-nodes-base.googleSheets, v4.5).  
    - Operation: `append`.  
    - Document ID: expression `{{ $json.logSheetId }}`.  
    - Sheet Name: expression `{{ $json.logSheetName }}`.  
    - Columns — mapping mode `defineBelow`:  
      - `Timestamp` → `{{ $json.timestamp }}`  
      - `Bot Answer` → `{{ $json.botAnswer }}`  
      - `Session ID` → `{{ $json.sessionId }}`  
      - `User Question` → `{{ $json.userQuestion }}`  
    - Options: `cellFormat` → `USER_ENTERED`.  
    - Connect the **same Google Sheets OAuth2 credential** used in step 4 (or create a new one if the log is in a different account).  
    - Connect: **8. Set — Prepare Log Fields → this node** (first output).

11. **Add a Set node — "10. Set — Return Answer to Chat"**  
    - Type: `Set` (n8n-nodes-base.set, v3.4).  
    - Assignments:  
      - `response` → `{{ $('8. Set — Prepare Log Fields').item.json.botAnswer }}`  
    - Connect: **8. Set — Prepare Log Fields → this node** (second output, parallel with step 10).

12. **Create your Google Sheets**  
    - **Knowledge Base sheet**: Create a Google Sheet. Add a tab named exactly `SEO FAQ`. In row 1, add headers: `Question`, `Answer`, `Category`, `Last Updated`. Fill in at least 5–10 Q&A rows for initial testing.  
    - **Chat Log sheet** (can be the same spreadsheet or a different one): Add a tab named exactly `Chat Log`. In row 1, add headers: `Session ID`, `Timestamp`, `User Question`, `Bot Answer`.

13. **Update configuration values**  
    - Open node **2. Set — Config Values**. Replace:  
      - `YOUR_KNOWLEDGE_BASE_SHEET_ID` with the ID from your knowledge base sheet URL (the string between `/d/` and `/edit`).  
      - `YOUR_CHAT_LOG_SHEET_ID` with the ID from your chat log sheet URL (same extraction method).  
      - `YOUR COMPANY NAME` with your actual company name (in both the `companyName` field and inside the `botPersona` string).

14. **Verify credentials**  
    - Open node **3. Google Sheets — Read Knowledge Base** → ensure Google Sheets OAuth2 is connected and authorised.  
    - Open node **9. Google Sheets — Log Chat** → connect the same (or a different) Google Sheets OAuth2 credential.  
    - Open node **6. OpenAI — GPT-4o-mini Model** → ensure your OpenAI API key is connected and has `gpt-4o-mini` access with available credits.

15. **Activate and test**  
    - Toggle the workflow to **Active**.  
    - Click on node **1. Chat Message Received** → copy the Chat URL.  
    - Open the URL in a browser → type a test question that exists in your SEO FAQ sheet → verify the response.  
    - Type a question NOT in the sheet → verify you receive the fallback message mentioning your company name.  
    - Check your **Chat Log** tab to confirm the exchange was logged.  
    - Test a follow-up question in the same session to confirm conversation memory works.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is designed for SEO agencies, consultants, and marketing teams who want a branded AI assistant without hallucinated or off-brand answers. | Workflow purpose |
| The chatbot reads the entire knowledge base on every question — no caching — so edits to the Google Sheet are reflected immediately on the next question without redeployment. | Live KB sync behaviour |
| GPT-4o-mini is explicitly instructed to never use outside knowledge. If it does stray, lower the temperature in node 6 from 0.3 to 0.1 for stricter responses. | Hallucination mitigation |
| Conversation memory is scoped to the browser session. Refreshing the page or closing the tab starts a new session and clears history. | Memory scope |
| The `sessionId` is generated from `$now.toMillis().toString()`. In extremely high-concurrency scenarios, consider appending a random suffix to avoid theoretical collisions. | Session ID uniqueness |
| The fallback message template includes `{{ $json.companyName }}`. If you change the variable name in the Set node, also update the AI Agent prompt text. | Consistency requirement |
| To expand the chatbot beyond SEO topics, edit the `botPersona` field in node 2 and populate your sheet with Q&A rows for the new domain (HR, product FAQs, onboarding, etc.). | Customisation |
| To increase answer length for complex topics, raise `maxTokens` in node 6 from 400 to ~700 and update the 150-word instruction in the AI Agent prompt. | Answer length adjustment |
| You can add a Slack notification after node 9 to alert a `#kb-gaps` channel when the fallback message is triggered, helping your team identify knowledge base gaps. | Enhancement suggestion |
| Incrementors — SEO agency that created and supports this workflow. Email: info@incrementors.com — Website: https://www.incrementors.com/ | Support & credits |
| The workflow's Overview sticky note contains a concise summary of all steps and setup instructions directly on the canvas for quick reference. | Canvas documentation |