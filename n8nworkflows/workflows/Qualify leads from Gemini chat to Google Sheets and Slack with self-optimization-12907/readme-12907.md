Qualify leads from Gemini chat to Google Sheets and Slack with self-optimization

https://n8nworkflows.xyz/workflows/qualify-leads-from-gemini-chat-to-google-sheets-and-slack-with-self-optimization-12907


# Qualify leads from Gemini chat to Google Sheets and Slack with self-optimization

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow qualifies inbound leads through an n8n Chat interface powered by **Google Gemini**, extracts structured lead fields (**name, industry, budget**), scores the lead, and then:
- logs the lead + AI suggestions to **Google Sheets**
- notifies Sales in **Slack** for “hot” leads
- runs a **daily self-optimization loop** that audits historical lead data and posts prompt-improvement recommendations to Slack.

**Target use cases:**
- Lead qualification from a website/chat bubble or embedded n8n chat
- Lightweight lead scoring and routing
- Continuous improvement of qualification prompts using performance feedback

### 1.1 Lead Intake & Conversational Qualification (Chat)
User chats → Gemini agent asks only for missing fields → maintains short-term memory.

### 1.2 Lead State Extraction & Validation
Parse the agent’s hidden JSON → validate completeness (“isReady”) → only proceed when qualified.

### 1.3 Scoring & Value Generation (“AI Council”)
Score the lead (hot vs not) → generate 3 concise growth tactics using Gemini.

### 1.4 Routing & Logging (Sheets + Slack)
Hot leads → log to Sheet1 + notify Slack.  
Not hot → log to Sheet2 (backup / lower-priority log).

### 1.5 Daily Self-Optimization Loop
Daily schedule → fetch historical lead rows → Gemini “auditor” proposes prompt/process improvements → send report to Slack.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Lead Intake & Conversational Qualification
**Overview:**  
Receives chat messages, uses a Gemini-powered agent to collect Name/Industry/Budget conversationally, and preserves short-term conversation context.

**Nodes involved:**
- **Lead Chat Trigger**
- **Chat Session Memory**
- **Chat Logic Brain**
- **Qualify Lead Conversationally**
- **Send Response to Chat Interface**

#### Node: Lead Chat Trigger
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — Entry point for n8n Chat.
- **Configuration (interpreted):**
  - `public: true` → the chat endpoint/UI is publicly accessible (depending on n8n deployment).
- **Connections:**
  - Output → **Qualify Lead Conversationally** (main)
- **Edge cases / failures:**
  - Public chat can be abused (spam, prompt injection, high usage). Consider rate limiting or authentication if deployed publicly.
  - If chat UI is not correctly configured in your n8n instance, messages may not arrive.

#### Node: Chat Session Memory
- **Type / role:** `memoryBufferWindow` — Keeps last N turns of conversation.
- **Configuration:**
  - `contextWindowLength: 4` → only the last 4 exchanges are remembered.
- **Connections:**
  - Output (ai_memory) → **Qualify Lead Conversationally** (ai_memory)
- **Edge cases / failures:**
  - If the lead qualification spans many turns, older critical details may fall out of memory; the agent may re-ask.
  - Memory node not connected → agent will not remember prior answers.

#### Node: Chat Logic Brain
- **Type / role:** `lmChatGoogleGemini` — Gemini chat model provider for LangChain nodes.
- **Configuration:**
  - Model: `models/gemini-1.5-flash` (fast/cheaper; can be less precise than “pro” models).
- **Connections:**
  - Output (ai_languageModel) → **Qualify Lead Conversationally**
  - Output (ai_languageModel) → **AI Council: Strategist**
- **Edge cases / failures:**
  - Invalid/expired Gemini API key or billing limits → authentication errors.
  - Model name mismatch vs account access → model-not-found/permission errors.
  - Safety filters or content policies may block certain responses.

#### Node: Qualify Lead Conversationally
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Conversational agent to gather fields.
- **Configuration:**
  - System message instructs:
    - goal: collect `name`, `industry`, `budget`
    - check chat history; don’t ask for already-known fields
    - **CRITICAL:** every reply must include a **hidden JSON block** at end:
      - Format: `{ 'name': '...', 'industry': '...', 'budget': ... }`
- **Key behaviors / variables:**
  - The downstream parser expects curly braces JSON-like content; single quotes are later converted to double quotes.
- **Connections:**
  - Input ← **Lead Chat Trigger** (main)
  - Input ← **Chat Session Memory** (ai_memory)
  - Input ← **Chat Logic Brain** (ai_languageModel)
  - Output → **Extract Lead State** (main)
  - Output → **Send Response to Chat Interface** (main)
- **Edge cases / failures:**
  - The agent may omit or malform the JSON block (common with LLMs), causing extraction to fail and lead to remain unqualified.
  - User may provide non-numeric budget (“ten thousand”); conversion to Number may result in `0`.
  - Prompt injection attempts may cause the agent to output unexpected JSON or additional braces.

#### Node: Send Response to Chat Interface
- **Type / role:** `respondToWebhook` — Returns the agent’s message to the chat client.
- **Configuration:** default response behavior.
- **Connections:**
  - Input ← **Qualify Lead Conversationally**
- **Edge cases / failures:**
  - If the workflow execution doesn’t reach this node (errors upstream), the chat UI may hang or show an error.
  - Ensure the node returns the correct field for the chat UI (depends on n8n chat trigger/response conventions).

---

### Block 2.2 — Lead State Extraction & Qualification Gate
**Overview:**  
Parses the hidden JSON emitted by the agent, normalizes values, sets `isReady`, and blocks downstream processing until all required fields are present.

**Nodes involved:**
- **Extract Lead State**
- **Check if Lead is Qualified**

#### Node: Extract Lead State
- **Type / role:** `code` — Parses/validates structured state.
- **Configuration (logic summary):**
  - Reads: `$node["Qualify Lead Conversationally"].json.output`
  - Regex extracts first `{ ... }` block (greedy across lines).
  - Replaces `'` with `"` then `JSON.parse`.
  - Builds state object with defaults:
    - `name: "unknown"`
    - `industry: "unknown"`
    - `budget: 0`
    - `isReady: false`
  - Sets `isReady = true` only if:
    - name != unknown
    - industry != unknown
    - budget > 0
- **Connections:**
  - Input ← **Qualify Lead Conversationally**
  - Output → **Check if Lead is Qualified**
- **Edge cases / failures:**
  - Regex `/\{[\s\S]*\}/` is greedy; if the agent outputs multiple `{}` blocks, it may capture too much and break JSON parsing.
  - Silent catch: parsing errors are swallowed (`catch (e) {}`), so failures degrade quietly to defaults and `isReady=false`.
  - Replacing all `'` with `"` can corrupt valid apostrophes inside values (e.g., `O'Reilly`) if included in JSON portion.

#### Node: Check if Lead is Qualified
- **Type / role:** `filter` — Gate that only passes qualified leads.
- **Configuration:**
  - Condition: `{{ $json.isReady }}` is `true`
  - Type validation: loose
- **Connections:**
  - Input ← **Extract Lead State**
  - Output → **Scoring Logic**
- **Edge cases / failures:**
  - If `isReady` is missing/non-boolean, “loose” validation may behave unexpectedly; but here it is always set by code.

---

### Block 2.3 — Scoring & AI Council Strategy Generation
**Overview:**  
Scores the lead based on budget and generates 3 growth tactics tailored to the industry and budget.

**Nodes involved:**
- **Scoring Logic**
- **AI Council: Strategist**
- **Is High ROI?**
- **Chat Logic Brain** (model provider connection used here)

#### Node: Scoring Logic
- **Type / role:** `code` — Adds `score` and `isHot`.
- **Configuration (logic summary):**
  - Reads data from **Extract Lead State** directly using:
    - `const data = $node["Extract Lead State"].json;`
  - Returns merged object:
    - `score: data.budget > 10000 ? 80 : 30`
    - `isHot: data.budget > 10000`
- **Connections:**
  - Input ← **Check if Lead is Qualified**
  - Output → **AI Council: Strategist**
- **Edge cases / failures:**
  - If budget is not numeric (0), lead will never be hot.
  - Hard threshold at 10,000 may not fit all industries; can be replaced by multi-factor scoring.

#### Node: AI Council: Strategist
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — LLM chain that generates growth tactics.
- **Configuration:**
  - Prompt (expression-based):
    - `As an AI Strategist, suggest 3 growth tactics for a company in the {{$json.industry}} industry with a budget of ${{$json.budget}}. Keep it concise.`
  - `hasOutputParser: true` (enabled; but no explicit parser is shown in JSON—behavior depends on node defaults/version).
- **Key expressions/variables:**
  - Uses `$json.industry` and `$json.budget` from **Scoring Logic** output.
- **Connections:**
  - Input ← **Scoring Logic**
  - Input (ai_languageModel) ← **Chat Logic Brain**
  - Output → **Is High ROI?**
- **Edge cases / failures:**
  - If industry remains “unknown” (should be filtered out earlier), output quality suffers.
  - Output field used later is `{{$json.text}}` (in Sheets nodes) and `...item.json.text` (in Slack node); ensure this node actually outputs `text` in your n8n version.

#### Node: Is High ROI?
- **Type / role:** `if` — Branching based on `isHot`.
- **Configuration:**
  - Condition checks:
    - left: `{{ $('Scoring Logic').item.json.isHot }}`
    - operation: boolean is true
- **Connections:**
  - Input ← **AI Council: Strategist**
  - **True branch** → **Log Lead to Spreadsheet** AND **Notify Sales of Hot Lead**
  - **False branch** → **Log Strategy to Backup**
- **Edge cases / failures:**
  - Uses `$('Scoring Logic').item.json.isHot` referencing another node’s item. If item linkage differs (multiple items, splits), this can break or reference wrong item.
  - If `isHot` is undefined, IF may route to false branch.

---

### Block 2.4 — Output: Google Sheets Logging + Slack Notifications
**Overview:**  
Writes the qualified lead and generated tactics into Google Sheets, and notifies Slack when the lead is hot.

**Nodes involved:**
- **Log Lead to Spreadsheet**
- **Log Strategy to Backup**
- **Notify Sales of Hot Lead**

#### Node: Log Lead to Spreadsheet
- **Type / role:** `googleSheets` — Append row for hot leads (primary log).
- **Configuration:**
  - Operation: **Append**
  - Document: `[YOUR_GOOGLE_SHEET_ID]`
  - Sheet tab: `Sheet1`
  - Column mapping (manual “define below”):
    - `Lead`:
      - `Name: {{ $('Extract Lead State').item.json.name }}`
      - `Industry: {{ $('Extract Lead State').item.json.industry }}`
      - `Budget: {{ $('Extract Lead State').item.json.budget }}`
    - `Suggestion`: `{{ $json.text }}` (expects strategist output in `text`)
- **Connections:**
  - Input ← **Is High ROI?** (true branch)
- **Edge cases / failures:**
  - Missing OAuth2 credentials or insufficient Sheet permissions → 401/403.
  - Spreadsheet ID placeholder must be replaced.
  - If headers don’t match exactly (`Lead`, `Suggestion`), append may fail or write into wrong columns (depending on node behavior/version).

#### Node: Notify Sales of Hot Lead
- **Type / role:** `slack` — Sends a channel message for hot leads.
- **Configuration:**
  - Channel: `[YOUR_SLACK_CHANNEL_ID]`
  - Message text:
    - `New High-Priority Lead: {{ name }}. Council Status: {{ strategist text }}.`
    - Specifically:
      - name from `$('Extract Lead State').item.json.name`
      - strategist output from `$('AI Council: Strategist').item.json.text`
  - `mrkdwn: false`
- **Connections:**
  - Input ← **Is High ROI?** (true branch)
- **Edge cases / failures:**
  - Bot not invited to channel → Slack API error.
  - Slack OAuth scopes missing (e.g., `chat:write`) → auth error.
  - If strategist node outputs `output` not `text`, message will be blank.

#### Node: Log Strategy to Backup
- **Type / role:** `googleSheets` — Append row for non-hot leads (secondary log).
- **Configuration:**
  - Operation: Append
  - Document: `[YOUR_GOOGLE_SHEET_ID]`
  - Sheet: `Sheet2`
  - Same column mapping as Sheet1.
- **Connections:**
  - Input ← **Is High ROI?** (false branch)
- **Edge cases / failures:**
  - Same as primary Sheets node; also ensure `Sheet2` exists.

---

### Block 2.5 — Daily Self-Optimization Loop (Audit + Slack Report)
**Overview:**  
Once per day, fetches historical lead data from Google Sheets, asks Gemini to identify friction/patterns and propose improvements, then posts the recommendations to Slack.

**Nodes involved:**
- **Daily Performance Audit Trigger**
- **Fetch Historical Lead Data**
- **Audit Analyst Brain**
- **Audit Lead Performance**
- **Send Optimization Report**

#### Node: Daily Performance Audit Trigger
- **Type / role:** `scheduleTrigger` — Time-based entry point.
- **Configuration:**
  - `rule.interval: [{}]` → appears under-specified in the exported JSON.
  - In practice, you must set a concrete schedule (e.g., every day at 09:00).
- **Connections:**
  - Output → **Fetch Historical Lead Data**
- **Edge cases / failures:**
  - If schedule isn’t configured properly, it may not run at all or may run too frequently.

#### Node: Fetch Historical Lead Data
- **Type / role:** `googleSheets` — Reads historical data for analysis.
- **Configuration:**
  - Document: `[YOUR_GOOGLE_SHEET_ID]`
  - Sheet: `Sheet1`
  - Operation not explicitly shown in parameters export; by position/name, it is intended to **read** rows.
- **Connections:**
  - Input ← **Daily Performance Audit Trigger**
  - Output → **Audit Lead Performance**
- **Edge cases / failures:**
  - Large sheets can cause slow reads/timeouts. Consider limits or filtering ranges.
  - Data format: downstream prompt expects a single `{{$json.text}}` field, but Sheets read typically outputs structured rows, not a `text` blob—this may require an additional formatting node in real usage.

#### Node: Audit Analyst Brain
- **Type / role:** `lmChatGoogleGemini` — Gemini model provider for auditing agent.
- **Configuration:**
  - Model: `models/gemini-1.5-flash`
- **Connections:**
  - Output (ai_languageModel) → **Audit Lead Performance**
- **Edge cases / failures:**
  - Same Gemini credential/model access issues as the Chat Logic Brain.

#### Node: Audit Lead Performance
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — An “auditor” agent that proposes improvements.
- **Configuration:**
  - Prompt text (expression):
    - `You are an AI System Auditor. Review the following lead data: {{ $json.text }}. Identify patterns and propose improvements.`
  - System message: “You are a helpful assistant”
  - Prompt type: define
- **Connections:**
  - Input ← **Fetch Historical Lead Data** (main)
  - Input (ai_languageModel) ← **Audit Analyst Brain**
  - Output → **Send Optimization Report**
- **Edge cases / failures:**
  - Potential mismatch: if Sheets output doesn’t provide `$json.text`, the auditor sees empty data. Typically you’d transform rows into a text summary first.
  - If the sheet contains sensitive data, ensure compliance and access controls.

#### Node: Send Optimization Report
- **Type / role:** `slack` — Posts audit findings to Slack.
- **Configuration:**
  - Channel: `[YOUR_SLACK_CHANNEL_ID]`
  - Message: `Optimization Report: Based on performance, update prompt to: {{ $json.output }}`
- **Connections:**
  - Input ← **Audit Lead Performance**
- **Edge cases / failures:**
  - Same Slack credential/channel membership/scope issues as the sales notification node.
  - If audit agent outputs in a different field than `output`, message may be empty.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Instructions | Sticky Note | Explains workflow purpose/setup/customization |  |  | How it works; setup; requirements; customization (Gemini/Sheets/Slack, scoring threshold, strategist prompt). |
| Lead Chat Trigger | LangChain Chat Trigger | Public chat entry point |  | Qualify Lead Conversationally | This section captures user input via chat and maintains a stateful conversation to qualify leads. |
| Chat Session Memory | Buffer Window Memory | Keeps last 4 turns of chat context |  | Qualify Lead Conversationally (ai_memory) | This section captures user input via chat and maintains a stateful conversation to qualify leads. |
| Chat Logic Brain | Gemini Chat Model | LLM provider for qualification + strategist |  | Qualify Lead Conversationally (ai_languageModel); AI Council: Strategist (ai_languageModel) | This section captures user input via chat and maintains a stateful conversation to qualify leads. |
| Qualify Lead Conversationally | LangChain Agent | Conversationally gathers name/industry/budget + emits hidden JSON | Lead Chat Trigger; Chat Session Memory; Chat Logic Brain | Extract Lead State; Send Response to Chat Interface | This section captures user input via chat and maintains a stateful conversation to qualify leads. |
| Send Response to Chat Interface | Respond to Webhook | Returns agent response to chat client | Qualify Lead Conversationally |  | This section captures user input via chat and maintains a stateful conversation to qualify leads. |
| Extract Lead State | Code | Parses hidden JSON into structured lead state | Qualify Lead Conversationally | Check if Lead is Qualified | Extracts JSON data from the chat and validates that all required fields are present before scoring. |
| Check if Lead is Qualified | Filter | Passes only qualified leads (isReady=true) | Extract Lead State | Scoring Logic | Extracts JSON data from the chat and validates that all required fields are present before scoring. |
| Scoring Logic | Code | Adds score + isHot based on budget threshold | Check if Lead is Qualified | AI Council: Strategist | Extracts JSON data from the chat and validates that all required fields are present before scoring. |
| AI Council: Strategist | LLM Chain | Produces 3 growth tactics based on industry/budget | Scoring Logic; Chat Logic Brain | Is High ROI? | Uses Gemini to generate industry-specific growth tactics for qualified leads. |
| Is High ROI? | IF | Routes hot vs non-hot | AI Council: Strategist | (true) Log Lead to Spreadsheet + Notify Sales of Hot Lead; (false) Log Strategy to Backup | Uses Gemini to generate industry-specific growth tactics for qualified leads. |
| Log Lead to Spreadsheet | Google Sheets | Append hot leads to Sheet1 | Is High ROI? (true) |  | Logs lead data to Google Sheets and alerts the team via Slack. |
| Notify Sales of Hot Lead | Slack | Notify channel about hot lead | Is High ROI? (true) |  | Logs lead data to Google Sheets and alerts the team via Slack. |
| Log Strategy to Backup | Google Sheets | Append non-hot leads to Sheet2 | Is High ROI? (false) |  | Logs lead data to Google Sheets and alerts the team via Slack. |
| Daily Performance Audit Trigger | Schedule Trigger | Daily entry point for optimization loop |  | Fetch Historical Lead Data | Feedback Loop: An autonomous loop that audits daily lead performance to suggest prompt improvements. |
| Fetch Historical Lead Data | Google Sheets | Read historical rows from Sheet1 | Daily Performance Audit Trigger | Audit Lead Performance | Feedback Loop: An autonomous loop that audits daily lead performance to suggest prompt improvements. |
| Audit Analyst Brain | Gemini Chat Model | LLM provider for auditing agent |  | Audit Lead Performance (ai_languageModel) | Feedback Loop: An autonomous loop that audits daily lead performance to suggest prompt improvements. |
| Audit Lead Performance | LangChain Agent | Analyzes lead data and proposes improvements | Fetch Historical Lead Data; Audit Analyst Brain | Send Optimization Report | Feedback Loop: An autonomous loop that audits daily lead performance to suggest prompt improvements. |
| Send Optimization Report | Slack | Posts optimization recommendations to Slack | Audit Lead Performance |  | Feedback Loop: An autonomous loop that audits daily lead performance to suggest prompt improvements. |
| Lead Ingestion Group | Sticky Note | Visual grouping note |  |  | This section captures user input via chat and maintains a stateful conversation to qualify leads. |
| Data Processing Group | Sticky Note | Visual grouping note |  |  | Extracts JSON data from the chat and validates that all required fields are present before scoring. |
| AI Council Group | Sticky Note | Visual grouping note |  |  | Uses Gemini to generate industry-specific growth tactics for qualified leads. |
| Output Group | Sticky Note | Visual grouping note |  |  | Logs lead data to Google Sheets and alerts the team via Slack. |
| Self-Optimization Group | Sticky Note | Visual grouping note |  |  | Feedback Loop: An autonomous loop that audits daily lead performance to suggest prompt improvements. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Qualify AI chat to Google Sheets and Slack with self-optimization*
- Ensure n8n version supports LangChain/AI nodes (as noted: n8n 1.0+; practically, use a recent 1.x).

### Phase A — Chat qualification path

2) **Add “Lead Chat Trigger”**
- Node type: **Chat Trigger (LangChain)**
- Set **Public = true** (or disable for controlled access)
- This is your main entry point.

3) **Add “Chat Session Memory”**
- Node type: **Buffer Window Memory**
- Set **Context Window Length = 4**
- Connect:
  - Memory node → **Qualify Lead Conversationally** via **ai_memory**

4) **Add “Chat Logic Brain”**
- Node type: **Google Gemini Chat Model**
- Credentials: **Google AI / Gemini API key**
- Model: `models/gemini-1.5-flash`
- Connect:
  - Chat Logic Brain → **Qualify Lead Conversationally** via **ai_languageModel**
  - Chat Logic Brain → **AI Council: Strategist** via **ai_languageModel**

5) **Add “Qualify Lead Conversationally”**
- Node type: **AI Agent (LangChain)**
- Configure **System Message** exactly (or equivalent):
  - Role: Lead Qualifier
  - Collect: Name, Industry, Budget
  - Don’t re-ask known info
  - Always append hidden JSON: `{ 'name': '...', 'industry': '...', 'budget': ... }`
- Connect:
  - **Lead Chat Trigger** → **Qualify Lead Conversationally** (main)

6) **Add “Send Response to Chat Interface”**
- Node type: **Respond to Webhook**
- Connect:
  - **Qualify Lead Conversationally** → **Send Response to Chat Interface** (main)

### Phase B — Parsing, validation, scoring, strategist

7) **Add “Extract Lead State”**
- Node type: **Code**
- Paste logic equivalent to:
  - read agent output
  - extract `{...}` block
  - replace `'` → `"`
  - parse JSON
  - set defaults + `isReady`
- Connect:
  - **Qualify Lead Conversationally** → **Extract Lead State** (main)

8) **Add “Check if Lead is Qualified”**
- Node type: **Filter**
- Condition: boolean true on `{{$json.isReady}}`
- Connect:
  - **Extract Lead State** → **Check if Lead is Qualified**

9) **Add “Scoring Logic”**
- Node type: **Code**
- Implement:
  - `score = 80` and `isHot=true` if budget > 10000 else `30/false`
  - Use the data from Extract Lead State
- Connect:
  - **Check if Lead is Qualified** → **Scoring Logic**

10) **Add “AI Council: Strategist”**
- Node type: **LLM Chain (LangChain)**
- Prompt text:
  - `As an AI Strategist, suggest 3 growth tactics for a company in the {{$json.industry}} industry with a budget of ${{$json.budget}}. Keep it concise.`
- Ensure it uses **Chat Logic Brain** as its language model via **ai_languageModel** connection.
- Connect:
  - **Scoring Logic** → **AI Council: Strategist**

11) **Add “Is High ROI?”**
- Node type: **IF**
- Condition: `{{ $('Scoring Logic').item.json.isHot }}` is true
- Connect:
  - **AI Council: Strategist** → **Is High ROI?**

### Phase C — Output routing (Sheets + Slack)

12) **Prepare credentials**
- **Google Sheets OAuth2** credential with access to the target spreadsheet
- **Slack OAuth2** credential with permission to post messages (`chat:write`) and bot invited to channel

13) **Create Google Sheet**
- Spreadsheet headers recommended by the sticky note: **Lead, Suggestion, Status**
- Create two tabs:
  - `Sheet1` (primary)
  - `Sheet2` (backup)

14) **Add “Log Lead to Spreadsheet” (hot path)**
- Node type: **Google Sheets**
- Operation: **Append**
- Document ID: your Spreadsheet ID
- Sheet: `Sheet1`
- Map columns:
  - Lead: formatted string using Extract Lead State fields
  - Suggestion: strategist output (ensure correct output field)
- Connect:
  - **Is High ROI? (true)** → **Log Lead to Spreadsheet**

15) **Add “Notify Sales of Hot Lead” (hot path)**
- Node type: **Slack**
- Send to channel: your channel ID
- Text: include lead name and strategist output
- Connect:
  - **Is High ROI? (true)** → **Notify Sales of Hot Lead**

16) **Add “Log Strategy to Backup” (non-hot path)**
- Node type: **Google Sheets**
- Operation: **Append**
- Document ID: same Spreadsheet ID
- Sheet: `Sheet2`
- Same mapping as primary
- Connect:
  - **Is High ROI? (false)** → **Log Strategy to Backup**

### Phase D — Daily self-optimization loop

17) **Add “Daily Performance Audit Trigger”**
- Node type: **Schedule Trigger**
- Configure a real schedule (e.g., daily at 08:00).
- Connect:
  - Trigger → **Fetch Historical Lead Data**

18) **Add “Fetch Historical Lead Data”**
- Node type: **Google Sheets**
- Configure to **Read/Get Many** rows from:
  - Document ID: same spreadsheet
  - Sheet: `Sheet1`
- Connect:
  - **Fetch Historical Lead Data** → **Audit Lead Performance**
- Note: You may need an extra formatting step (e.g., “Code” or “Aggregate”) to convert rows into a single text summary if your agent expects `$json.text`.

19) **Add “Audit Analyst Brain”**
- Node type: **Google Gemini Chat Model**
- Model: `models/gemini-1.5-flash`
- Credentials: Gemini API key
- Connect:
  - Audit Analyst Brain → **Audit Lead Performance** via **ai_languageModel**

20) **Add “Audit Lead Performance”**
- Node type: **AI Agent (LangChain)**
- Prompt: “AI System Auditor…” referencing the lead data input
- Connect:
  - **Fetch Historical Lead Data** → **Audit Lead Performance** (main)

21) **Add “Send Optimization Report”**
- Node type: **Slack**
- Channel: same (or separate ops channel)
- Text uses audit output field (confirm if it is `output`)
- Connect:
  - **Audit Lead Performance** → **Send Optimization Report**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Credentials: Connect Google Gemini (API Key), Google Sheets (OAuth2), Slack (OAuth2). | From “Main Instructions” sticky note |
| Google Sheets: Create headers **Lead, Suggestion, Status**; copy Spreadsheet ID into Sheets nodes. | From “Main Instructions” sticky note |
| Slack: Invite your n8n bot to the channel (e.g., `/invite @n8n`) and select that channel in Slack nodes. | From “Main Instructions” sticky note |
| Memory: Ensure Buffer Window Memory is connected to the AI Agent to maintain conversation state. | From “Main Instructions” sticky note |
| Scoring Logic: Hot lead is currently `budget > 10,000`; adjust code to change threshold/criteria. | From “Main Instructions” sticky note |
| AI Strategist prompt can be modified to change value offered (e.g., free audits instead of growth tactics). | From “Main Instructions” sticky note |