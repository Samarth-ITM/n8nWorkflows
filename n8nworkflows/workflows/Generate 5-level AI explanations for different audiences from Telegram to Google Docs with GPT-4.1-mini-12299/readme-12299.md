Generate 5-level AI explanations for different audiences from Telegram to Google Docs with GPT-4.1-mini

https://n8nworkflows.xyz/workflows/generate-5-level-ai-explanations-for-different-audiences-from-telegram-to-google-docs-with-gpt-4-1-mini-12299


# Generate 5-level AI explanations for different audiences from Telegram to Google Docs with GPT-4.1-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Generate 5-level AI explanations from Telegram to Google Docs  
**Purpose:** When a user sends a question to a Telegram bot, the workflow generates **five AI explanations** at different comprehension levels (5-year-old, teenager, graduate, PhD, business), then:
- Sends **6 Telegram messages** (1 header + 5 explanations)
- Archives the combined output into a **Google Doc** for reference

**Primary use cases**
- Education/content repurposing across audiences
- Rapid “multi-depth” summaries for internal/external communications
- Knowledge accessibility (from child-friendly to executive-ready framing)

### 1.1 Input Reception (Telegram → normalized fields)
Receives the Telegram message, extracts the question text, chat ID, and a timestamp.

### 1.2 Item Fan-Out (create 5 items for parallel levels)
Creates 5 items (one per level) containing prompt metadata and the same user query.

### 1.3 Routing (Switch by level)
Routes each item to the corresponding AI agent prompt configuration.

### 1.4 AI Generation (5 parallel agents on GPT-4.1-mini)
Runs 5 agent nodes (one per audience), all using the same OpenAI chat model node.

### 1.5 Response Normalization (content extraction per branch)
Each branch post-processes the agent output into a normalized structure `{level, title, emoji, content, chatId, query, timestamp}`.

### 1.6 Merge Strategy (binary merges → all five)
Merges responses in a “2-at-a-time” pattern to reduce dropped items vs. merging 5 streams at once.

### 1.7 Aggregation & Formatting (Telegram + Google Docs)
Aggregates exactly 5 responses, sorts them in the intended order, formats:
- 6 HTML Telegram messages with safe escaping and truncation safeguards
- Clean plain text for Google Docs insertion

### 1.8 Delivery & Archival
Sends Telegram messages to the user and inserts the formatted output into a Google Doc.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Listens for Telegram messages and extracts the minimum fields needed for downstream processing (query, chatId, timestamp).  
**Nodes involved:** Receive User Question → Extract Query Data

#### Node: Receive User Question
- **Type / role:** `telegramTrigger` — Entry point trigger (listens for new messages)
- **Configuration:** Subscribes to `message` updates.
- **Input/Output:**
  - **Input:** Telegram event
  - **Output:** Telegram message payload (`$json.message.text`, `$json.message.chat.id`, etc.)
- **Failure/edge cases:**
  - Bot token missing/invalid (credentials error)
  - Telegram webhook/polling misconfigured
  - Non-text messages (photos/voice) may not have `message.text`

#### Node: Extract Query Data
- **Type / role:** `set` — Normalizes fields into a simple schema
- **Key fields created:**
  - `userQuery = {{$json.message.text}}`
  - `chatId = {{$json.message.chat.id}}` (stored as string)
  - `timestamp = {{$now.toISO()}}`
- **Input/Output:**  
  - Input: Telegram trigger JSON  
  - Output: `{ userQuery, chatId, timestamp }`
- **Failure/edge cases:**
  - If message is not text, `message.text` is undefined → downstream AI may get empty query unless validated.

---

### Block 2 — Item Fan-Out (Create 5 level items)
**Overview:** Produces 5 items from the single Telegram question, each tagged with a target audience “level” and a dedicated system prompt.  
**Nodes involved:** Route to Appropriate Level (Code)

#### Node: Route to Appropriate Level
- **Type / role:** `code` — Duplicates input into 5 items for parallel processing
- **Configuration choices (interpreted):**
  - Reads the first input item: `const input = $input.first().json;`
  - Creates a `levels` array with 5 objects:
    - `level` identifiers: `5-year-old`, `teenager`, `graduate`, `phd`, `business`
    - `emoji`, `title`, and `systemPrompt` per level
  - Returns `levels.map(...)` to emit 5 items with:
    - `level, emoji, title, systemPrompt, query, chatId, timestamp`
- **Connections:**  
  - Input: Extract Query Data  
  - Output: Switch
- **Failure/edge cases:**
  - Missing `userQuery` produces empty `query`
  - Large prompts increase token usage and latency
  - Any runtime error in JS stops the workflow

---

### Block 3 — Routing
**Overview:** Routes each of the 5 items to the matching AI agent path by checking `$json.level`.  
**Nodes involved:** Switch

#### Node: Switch
- **Type / role:** `switch` — Conditional branching based on `level`
- **Rules (5 outputs):**
  - Output 0: if `{{$json.level}} == "5-year-old"`
  - Output 1: if `{{$json.level}} == "teenager"`
  - Output 2: if `{{$json.level}} == "graduate"`
  - Output 3: if `{{$json.level}} == "phd"`
  - Output 4: if `{{$json.level}} == "business"`
- **Connections:**
  - Input: Route to Appropriate Level
  - Outputs:
    - 5-Year-Old Story Mode
    - Teenager Level
    - Graduate Level
    - PhD Research Level
    - Business Executive Level
- **Failure/edge cases:**
  - If `level` is missing or unexpected, the item won’t match any rule (and is effectively dropped). No default route is configured.

---

### Block 4 — AI Generation (5 parallel audience explanations)
**Overview:** Uses a shared OpenAI chat model (gpt-4.1-mini) with five agent nodes, each having a different system message tailored to an audience.  
**Nodes involved:** OpenAI Chat Model + (5 agent nodes)

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — Provides the LLM to agent nodes
- **Configuration:** Model set to **`gpt-4.1-mini`**
- **Connections:** Provides `ai_languageModel` to all five agents.
- **Failure/edge cases:**
  - Missing OpenAI credentials / invalid API key
  - Rate limiting / quota exceeded
  - Model name not available in the account/region
  - Latency/timeouts on long prompts

#### Node: 5-Year-Old Story Mode
- **Type / role:** `langchain.agent` — Generates child-friendly short story explanation
- **Configuration choices:**
  - Uses a detailed **system message** emphasizing story format, emojis, simple vocabulary, no default “AI/robots”.
  - Prompt includes `Question to explain: {{ $json.query }}`
  - `promptType: define` (prompt is explicitly provided in node)
- **Input/Output:**
  - Input: Switch (5-year-old item)
  - Output: 5-Year-Old (Code) for normalization
- **Failure/edge cases:**
  - Output field shape can vary by agent version; handled downstream by extraction code.

#### Node: Teenager Level
- **Type / role:** `langchain.agent` — Relatable teen explanation
- **Configuration:**
  - `text = {{$json.query}}`
  - System message focuses on casual but not “cringe” tone, relevance to teen life.
- **Connections:** Switch → Teenager Level → Teenager (Code)
- **Failure/edge cases:** same as above (LLM/agent output variability, rate limits)

#### Node: Graduate Level
- **Type / role:** `langchain.agent` — Professional, technical depth without over-specialization
- **Configuration:** Uses academic framing, definitions, mechanisms, applications.
- **Connections:** Switch → Graduate Level → Graduate (Code)

#### Node: PhD Research Level
- **Type / role:** `langchain.agent` — Expert-level rigorous explanation
- **Configuration:** Emphasizes formal definitions, research state, limitations, math where relevant.
- **Connections:** Switch → PhD Research Level → PhD (Code)

#### Node: Business Executive Level
- **Type / role:** `langchain.agent` — Executive framing (ROI, KPIs, strategy)
- **Configuration notes:**
  - The node’s `text` uses: `{{ $('Extract Query Data').item.json.userQuery }}`
    - This is a **cross-node reference** rather than `{{$json.query}}`.
  - System message expects `{{ $json.query }}` inside it as well.
- **Connections:** Switch → Business Executive Level → Business (Code)
- **Edge cases / potential bug:**
  - If the agent receives an item without the expected context, cross-node `$('Extract Query Data')...` can be brittle when running with multiple items or different execution contexts.
  - Safer would be `text: {{$json.query}}` (consistent with other levels), since the fan-out already injects `query`.

**Version-specific requirements (AI nodes):**
- These nodes come from **`@n8n/n8n-nodes-langchain`**; ensure the LangChain nodes package is installed and compatible with your n8n version.
- Agent output fields (`output`, `text`, `message.content`) may vary between node versions; normalization code accounts for this.

---

### Block 5 — Response Normalization / “Error Handler” Extraction
**Overview:** Each branch converts the agent output into a consistent JSON structure and re-attaches original metadata by looking up the corresponding “level” item from the fan-out node.  
**Nodes involved:** 5-Year-Old, Teenager, Graduate, PhD, Business (all Code)

Common behavior across all 5 nodes:
- **Type / role:** `code` — Extract content from agent output using fallbacks:
  - `responseItem.output || responseItem.text || responseItem.message?.content || "No response generated"`
- **Rehydrate metadata:** Finds the original fan-out item:
  - `$("Route to Appropriate Level").all().find(item => item.json.level === "<level>")`
- **Outputs:** single item `{ level, emoji, title, content, chatId, query, timestamp }`

#### Node: 5-Year-Old
- **Level:** `5-year-old`
- **Edge cases:**
  - If `$("Route to Appropriate Level").all()` is not accessible due to execution settings or partial runs, `originalItem` may be undefined → chatId/query fallback to `"unknown"`.

#### Node: Teenager
- Same pattern for `teenager`

#### Node: Graduate
- Same pattern for `graduate`

#### Node: PhD
- Same pattern for `phd`

#### Node: Business
- Same pattern for `business`

**General failure modes for this block:**
- Cross-node `.all()` access can fail in certain execution contexts (e.g., if nodes are executed in isolation, or data is not retained).
- If the agent returns a different structure, `content` may become `"No response generated"` (but workflow continues).

---

### Block 6 — Merge Strategy (Binary tree)
**Overview:** Merges the five normalized results in a staged approach to improve reliability (reduces chance of missing items).  
**Nodes involved:** Child + Teen → Grad + PhD → First 4 Levels → All 5 Levels

#### Node: Child + Teen
- **Type / role:** `merge` — Combines two inputs
- **Inputs:**  
  - Input 0: 5-Year-Old (Code)  
  - Input 1: Teenager (Code)
- **Output:** Combined stream to First 4 Levels

#### Node: Grad + PhD
- **Type / role:** `merge`
- **Inputs:**
  - Input 0: Graduate (Code)
  - Input 1: PhD (Code)
- **Output:** Combined stream to First 4 Levels

#### Node: First 4 Levels
- **Type / role:** `merge`
- **Inputs:**
  - Input 0: Child + Teen
  - Input 1: Grad + PhD
- **Output:** Combined 4 items to All 5 Levels

#### Node: All 5 Levels
- **Type / role:** `merge`
- **Inputs:**
  - Input 0: First 4 Levels
  - Input 1: Business (Code)
- **Output:** Combined 5 items to aggregation

**Failure/edge cases:**
- Merge node mode isn’t explicitly shown; defaults can differ. If configured incorrectly (e.g., “Wait” vs “Append”), could change behavior. The intent here is to **collect/append items**.
- If any upstream branch fails entirely, aggregation later throws because it expects exactly 5 items.

---

### Block 7 — Aggregation & Final Output Formatting
**Overview:** Validates all five responses exist, sorts them, then formats for Telegram (HTML) and Google Docs (plain text).  
**Nodes involved:** Aggregate & Structure All Responses → Format 6 Messages for Telegram → Send to User via Telegram; and Aggregate → Format Plain Text for Docs → Archive to Google Docs

#### Node: Aggregate & Structure All Responses
- **Type / role:** `code` — Aggregates and validates item count
- **Logic highlights:**
  - Collects `$input.all()` and extracts `item.json`
  - **Hard validation:** `if (allItems.length !== 5) throw new Error(...)`
  - Sort order: `["5-year-old","teenager","graduate","phd","business"]`
  - Outputs one item:
    - `responses: sortedItems`
    - `chatId/query/timestamp` taken from the first sorted item
- **Connections:** Input from All 5 Levels; outputs to both formatting nodes
- **Failure/edge cases:**
  - Any missing branch causes a hard failure
  - If any item lacks `level`, sorting may misbehave (indexOf returns -1)

#### Node: Format 6 Messages for Telegram
- **Type / role:** `code` — Produces 6 outbound Telegram message payloads
- **Key behaviors:**
  - Creates 1 header message + 5 messages for each response
  - Uses Telegram `parse_mode: HTML`
  - Includes:
    - Markdown “cleanup” step (`cleanMarkdown`) converting common markdown patterns to HTML-like tags
    - HTML escaping step (`escapeHtml`)
    - “Smart truncate” to ~4000 chars (Telegram hard limit ~4096)
- **Important implementation note (potential formatting issue):**
  - The code converts markdown to `<b>/<i>` tags, **then escapes HTML**, which turns `<b>` into `&lt;b&gt;` and will display literally (no bold/italic).
  - If the goal is real HTML formatting, escaping should happen **before** inserting `<b>/<i>`, or escape selectively while allowing known tags.
- **Outputs:** 6 items each with `{chatId, text, messageNumber}`
- **Failure/edge cases:**
  - Very long content is truncated
  - If response contains characters needing escaping, current logic makes it safe but may disable intended HTML tags

#### Node: Send to User via Telegram
- **Type / role:** `telegram` — Sends messages to the originating chat
- **Configuration:**
  - `chatId = {{$json.chatId}}`
  - `text = {{$json.text}}`
  - `parse_mode = HTML`
  - `appendAttribution = false`
- **Input/Output:** Consumes the 6 formatted items; sends 6 messages
- **Failure/edge cases:**
  - Invalid chatId (if “unknown”)
  - Telegram API rate limits (burst of 6 messages)
  - HTML parse errors if content includes unsupported tags/entities

#### Node: Format Plain Text for Docs
- **Type / role:** `code` — Builds a clean plain text archive
- **Logic:**
  - Creates a header with separators and date
  - For each response: prints emoji, uppercase title, divider line, then content
  - Outputs `{ docsContent }`
- **Failure/edge cases:**
  - Large insert size may exceed Google Docs API limits for a single update (depends on content length)

#### Node: Archive to Google Docs
- **Type / role:** `googleDocs` — Inserts text into a Google Document
- **Configuration:**
  - Operation: `update`
  - Action: `insert` with `text = {{$json.docsContent}}`
  - Document URL placeholder: `REPLACE_WITH_YOUR_GOOGLE_DOC_URL`
- **Failure/edge cases:**
  - Missing/invalid Google credentials
  - Document URL not shared/accessible to the authenticated account
  - Insert permissions issue (doc is read-only)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Comment: Input layer description |  |  | ## Input Layer<br><br>**1. Telegram receives your question**<br>**2. Extracts your chat ID and question**<br>**3. Creates 5 copies (one per level)** |
| Sticky Note2 | stickyNote | Comment/blank area |  |  |  |
| Sticky Note3 | stickyNote | Comment/blank area |  |  |  |
| Sticky Note4 | stickyNote | Comment: merge strategy rationale |  |  | ## Merge Strategy<br><br>**Combines responses 2 at a time:**<br>**1+2 → 3+4 → Combine → Add 5th**<br><br>**Why not merge all 5 at once?**<br>**More reliable. Never loses responses.** |
| Sticky Note5 | stickyNote | Comment: Telegram output |  |  | ## Telegram Output<br><br>**Sends you 6 messages:**<br><br>**1. Header (your question + date)**<br>**2-6. Five explanations** |
| Sticky Note6 | stickyNote | Comment: Google Docs output |  |  | ## Google Doc Output  (for reference)<br><br>**Sends you the final output with 5 variations:** |
| Sticky Note7 | stickyNote | Comment: try it out, overview, setup links |  |  | # 🚀 Try It Out<br><br>Send a message to your Telegram bot with any question (e.g., "What is machine learning?"). Within seconds, you'll receive 6 messages: a header plus 5 AI-generated explanations tailored for different audiences.<br><br># 📖 Overview<br><br>The AI Knowledge Spectrum transforms complex topics into 5 comprehension levels: child-friendly stories (5-year-old), relatable teen explanations, college graduate depth, PhD-level analysis, and business-focused insights. Perfect for educators, content creators, and anyone making knowledge accessible.<br><br># ⚙️ How It Works<br><br>1. Input: Telegram receives question → Extract chatId, query, timestamp<br>2. Route: Create 5 items → Switch routes each to AI agent<br>3. Process: 5 AI agents run in parallel (5YO, teen, grad, PhD, business)<br>4. Handle: Error handlers ensure all responses complete<br>5. Merge: Binary tree (4 merges) → Combines all 5 reliably<br>6. Aggregate: Sort by level → Structure output<br>7. Format: Create 6 messages (header + 5 levels) with HTML<br>8. Send: Deliver to Telegram chat<br>9. Archive: Append to Google Docs<br><br># 🔧 Setup Requirements<br><br>1. Add Telegram bot token<br>2. Add OpenAI API key<br>3. Connect Google account<br>4. Create Google doc<br>5. Configure Workflow<br>6. Activate & Test<br><br># ❓ Need Help?<br><br>For detailed notes and implementation, please leverage the README document at<br>https://drive.google.com/file/d/19Fx-FoihL70qpOi4CnEwQ6Sud2dbUnE_/view?usp=sharing<br><br>Join the n8n community forum (https://community.n8n.io/) for support<br><br>**Creates 5 parallel paths.** |
| Sticky Note8 | stickyNote | Comment/blank area |  |  |  |
| Sticky Note9 | stickyNote | Comment/blank area |  |  |  |
| Sticky Note10 | stickyNote | Comment/blank area |  |  |  |
| Sticky Note11 | stickyNote | Comment/blank area |  |  |  |
| Sticky Note12 | stickyNote | Comment/blank area |  |  |  |
| Sticky Note13 | stickyNote | Comment/blank area |  |  |  |
| Sticky Note14 | stickyNote | Comment/blank area |  |  |  |
| Sticky Note15 | stickyNote | Comment: aggregation/formatting |  |  | ## Collection of responses  & Formatting<br><br>**Combines all five responses into a single result**<br><br>**Organizes the data & formats it for final output** |
| Sticky Note16 | stickyNote | Comment: AI processing section label |  |  | ## AI Processing & Error Handling |
| Receive User Question | telegramTrigger | Receive Telegram message |  | Extract Query Data | ## Input Layer<br><br>**1. Telegram receives your question**<br>**2. Extracts your chat ID and question**<br>**3. Creates 5 copies (one per level)** |
| Extract Query Data | set | Extract query/chatId/timestamp | Receive User Question | Route to Appropriate Level | ## Input Layer<br><br>**1. Telegram receives your question**<br>**2. Extracts your chat ID and question**<br>**3. Creates 5 copies (one per level)** |
| Route to Appropriate Level | code | Create 5 level items | Extract Query Data | Switch | ## Input Layer<br><br>**1. Telegram receives your question**<br>**2. Extracts your chat ID and question**<br>**3. Creates 5 copies (one per level)** |
| Switch | switch | Route by level | Route to Appropriate Level | 5-Year-Old Story Mode; Teenager Level; Graduate Level; PhD Research Level; Business Executive Level | ## Routing<br><br>**Routes items to AI agents by understanding level.** |
| OpenAI Chat Model | lmChatOpenAi | Shared LLM (gpt-4.1-mini) |  | 5 agent nodes (as AI language model) | ## AI Processing & Error Handling |
| 5-Year-Old Story Mode | langchain.agent | Generate child story explanation | Switch | 5-Year-Old | ## AI Processing & Error Handling |
| Teenager Level | langchain.agent | Generate teen explanation | Switch | Teenager | ## AI Processing & Error Handling |
| Graduate Level | langchain.agent | Generate graduate explanation | Switch | Graduate | ## AI Processing & Error Handling |
| PhD Research Level | langchain.agent | Generate PhD explanation | Switch | PhD | ## AI Processing & Error Handling |
| Business Executive Level | langchain.agent | Generate business explanation | Switch | Business | ## AI Processing & Error Handling |
| 5-Year-Old | code | Normalize/extract response for 5YO | 5-Year-Old Story Mode | Child + Teen |  |
| Teenager | code | Normalize/extract response for teen | Teenager Level | Child + Teen |  |
| Graduate | code | Normalize/extract response for grad | Graduate Level | Grad + PhD |  |
| PhD | code | Normalize/extract response for PhD | PhD Research Level | Grad + PhD |  |
| Business | code | Normalize/extract response for business | Business Executive Level | All 5 Levels |  |
| Child + Teen | merge | Merge 5YO + teen | 5-Year-Old; Teenager | First 4 Levels | ## Merge Strategy<br><br>**Combines responses 2 at a time:**<br>**1+2 → 3+4 → Combine → Add 5th**<br><br>**Why not merge all 5 at once?**<br>**More reliable. Never loses responses.** |
| Grad + PhD | merge | Merge grad + PhD | Graduate; PhD | First 4 Levels | ## Merge Strategy<br><br>**Combines responses 2 at a time:**<br>**1+2 → 3+4 → Combine → Add 5th**<br><br>**Why not merge all 5 at once?**<br>**More reliable. Never loses responses.** |
| First 4 Levels | merge | Merge first two merged pairs | Child + Teen; Grad + PhD | All 5 Levels | ## Merge Strategy<br><br>**Combines responses 2 at a time:**<br>**1+2 → 3+4 → Combine → Add 5th**<br><br>**Why not merge all 5 at once?**<br>**More reliable. Never loses responses.** |
| All 5 Levels | merge | Add business to first four | First 4 Levels; Business | Aggregate & Structure All Responses | ## Merge Strategy<br><br>**Combines responses 2 at a time:**<br>**1+2 → 3+4 → Combine → Add 5th**<br><br>**Why not merge all 5 at once?**<br>**More reliable. Never loses responses.** |
| Aggregate & Structure All Responses | code | Validate=5, sort, wrap | All 5 Levels | Format 6 Messages for Telegram; Format Plain Text for Docs | ## Collection of responses  & Formatting<br><br>**Combines all five responses into a single result**<br><br>**Organizes the data & formats it for final output** |
| Format 6 Messages for Telegram | code | Build header + 5 HTML messages | Aggregate & Structure All Responses | Send to User via Telegram | ## Telegram Output<br><br>**Sends you 6 messages:**<br><br>**1. Header (your question + date)**<br>**2-6. Five explanations** |
| Send to User via Telegram | telegram | Send messages to Telegram chat | Format 6 Messages for Telegram |  | ## Telegram Output<br><br>**Sends you 6 messages:**<br><br>**1. Header (your question + date)**<br>**2-6. Five explanations** |
| Format Plain Text for Docs | code | Build plain text archive | Aggregate & Structure All Responses | Archive to Google Docs | ## Google Doc Output  (for reference)<br><br>**Sends you the final output with 5 variations:** |
| Archive to Google Docs | googleDocs | Insert into Google Doc | Format Plain Text for Docs |  | ## Google Doc Output  (for reference)<br><br>**Sends you the final output with 5 variations:** |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger: Telegram**
   1. Add node **Telegram Trigger**
   2. Name: `Receive User Question`
   3. Updates: **message**
   4. **Credentials:** Telegram Bot Token (create a bot via BotFather, paste token)

2) **Extract key fields**
   1. Add node **Set**
   2. Name: `Extract Query Data`
   3. Add fields:
      - `userQuery` (String) = `{{$json.message.text}}`
      - `chatId` (String) = `{{$json.message.chat.id}}`
      - `timestamp` (String) = `{{$now.toISO()}}`
   4. Connect: `Receive User Question` → `Extract Query Data`

3) **Fan-out into 5 items**
   1. Add node **Code**
   2. Name: `Route to Appropriate Level`
   3. Paste logic that returns 5 items with:
      - `level` in: `5-year-old`, `teenager`, `graduate`, `phd`, `business`
      - `emoji`, `title`, `systemPrompt`
      - `query`, `chatId`, `timestamp` from the Set node output
   4. Connect: `Extract Query Data` → `Route to Appropriate Level`

4) **Route each item by level**
   1. Add node **Switch**
   2. Name: `Switch`
   3. Add 5 rules (string equals) on `={{$json.level}}` matching the 5 level values
   4. Connect: `Route to Appropriate Level` → `Switch`

5) **Add the shared OpenAI model**
   1. Add node **OpenAI Chat Model** (LangChain)
   2. Name: `OpenAI Chat Model`
   3. Model: **gpt-4.1-mini**
   4. **Credentials:** OpenAI API key
   5. This node must connect as **AI Language Model** to each agent node (not via “main”).

6) **Create 5 agent nodes (one per audience)**
   For each agent:
   1. Add node **AI Agent** (LangChain Agent)
   2. Set `promptType` to “Define” and provide a tailored **system message**
   3. Set `text` to the user question (recommended: `{{$json.query}}` for consistency)
   4. Connect the agent’s **AI language model** input to `OpenAI Chat Model`
   5. Connect Switch outputs:
      - Output “5-year-old” → `5-Year-Old Story Mode`
      - Output “teenager” → `Teenager Level`
      - Output “graduate” → `Graduate Level`
      - Output “phd” → `PhD Research Level`
      - Output “business” → `Business Executive Level`

7) **Normalize each agent response**
   For each branch, add a **Code** node:
   - Names: `5-Year-Old`, `Teenager`, `Graduate`, `PhD`, `Business`
   - Logic:
     - Pull agent output from `$input.first().json`
     - Extract content via fallback: `output || text || message.content`
     - Reattach `chatId/query/timestamp` by looking up the matching item from `Route to Appropriate Level`
   - Connect each agent → its code node.

8) **Merge in a binary tree**
   1. Add **Merge** node: `Child + Teen`
      - Input 1: `5-Year-Old`
      - Input 2: `Teenager`
   2. Add **Merge** node: `Grad + PhD`
      - Input 1: `Graduate`
      - Input 2: `PhD`
   3. Add **Merge** node: `First 4 Levels`
      - Inputs: `Child + Teen` and `Grad + PhD`
   4. Add **Merge** node: `All 5 Levels`
      - Inputs: `First 4 Levels` and `Business`

9) **Aggregate, validate, and sort**
   1. Add **Code** node: `Aggregate & Structure All Responses`
   2. Implement:
      - Expect exactly 5 items; throw if not
      - Sort by level order
      - Output single item with `responses`, `chatId`, `query`, `timestamp`
   3. Connect: `All 5 Levels` → `Aggregate & Structure All Responses`

10) **Format Telegram messages**
   1. Add **Code** node: `Format 6 Messages for Telegram`
   2. Produce 6 items:
      - 1 header + 5 response messages
      - Each item must include `{chatId, text}`
   3. Connect: `Aggregate & Structure All Responses` → `Format 6 Messages for Telegram`

11) **Send to Telegram**
   1. Add node **Telegram**
   2. Name: `Send to User via Telegram`
   3. Set:
      - Chat ID = `{{$json.chatId}}`
      - Text = `{{$json.text}}`
      - Parse mode = **HTML**
      - Append attribution = false
   4. Connect: `Format 6 Messages for Telegram` → `Send to User via Telegram`
   5. **Credentials:** Same Telegram bot credentials as trigger (or another Telegram credential)

12) **Format for Google Docs**
   1. Add **Code** node: `Format Plain Text for Docs`
   2. Build a single string `docsContent` containing question/date and all 5 responses
   3. Connect: `Aggregate & Structure All Responses` → `Format Plain Text for Docs`

13) **Archive to Google Docs**
   1. Add node **Google Docs**
   2. Name: `Archive to Google Docs`
   3. Operation: **Update**
   4. Action: **Insert text**
      - Text = `{{$json.docsContent}}`
   5. Document URL: set to your document link (replace placeholder)
   6. **Credentials:** Google OAuth2 account with access to the doc
   7. Connect: `Format Plain Text for Docs` → `Archive to Google Docs`

14) **Activate and test**
   - Activate workflow
   - Send a text message to your bot
   - Verify: 6 Telegram messages + appended Google Docs entry

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| README / detailed notes referenced in the canvas | https://drive.google.com/file/d/19Fx-FoihL70qpOi4CnEwQ6Sud2dbUnE_/view?usp=sharing |
| n8n community forum for support | https://community.n8n.io/ |
| Design intent: “Creates 5 parallel paths” and merges 2-at-a-time for reliability | Included in Sticky Note7 and Sticky Note4 |
| Setup requirements: Telegram token, OpenAI key, Google account + target Doc URL | Included in Sticky Note7 |

