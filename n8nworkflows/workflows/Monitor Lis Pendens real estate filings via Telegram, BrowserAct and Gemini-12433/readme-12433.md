Monitor Lis Pendens real estate filings via Telegram, BrowserAct and Gemini

https://n8nworkflows.xyz/workflows/monitor-lis-pendens-real-estate-filings-via-telegram--browseract-and-gemini-12433


# Monitor Lis Pendens real estate filings via Telegram, BrowserAct and Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Monitor Lis Pendens real estate filings via Telegram, BrowserAct and Gemini  
**Workflow name (n8n):** Monitor real estate filings via Telegram, BrowserAct and Gemini

**Purpose:**  
This workflow receives messages from a Telegram bot, classifies the intent (real estate/Lis Pendens request vs. casual chat), and then either:
- **Real estate branch:** Automatically computes a 5‑day date range, runs a BrowserAct extraction workflow to scrape Lis Pendens filings, uses an LLM (OpenRouter GPT‑4.1) to format results into Telegram-safe HTML chunks (≤ 3500 chars), then sends one or multiple messages back to Telegram with rate-limit protection.
- **Chat branch:** Uses Gemini to generate a conversational reply and sends it back on Telegram.

**Target use cases:**
- On-demand monitoring of recent Lis Pendens filings (e.g., last 5 days) via a Telegram bot.
- A single entry point that also supports general conversation without running scraping.

### 1.1 Input Reception (Telegram Trigger)
Receives inbound Telegram messages and forwards the text to the classifier agent.

### 1.2 Intent Classification (LLM + Structured Output)
Uses OpenRouter GPT‑4.1 with a structured output parser to label the message as either `Real_State` or `chat`.

### 1.3 Real Estate Extraction Pipeline (Date range → BrowserAct)
Computes a dynamic date range (today and last 5 days), formats dates to US locale strings, merges both date paths, and passes them into a BrowserAct workflow (template-based automated browsing/scraping).

### 1.4 AI Formatting for Telegram + Delivery
Formats the scraped JSON into Telegram HTML blocks, splits into multiple messages if needed, iterates messages with waiting to avoid rate limits, and sends to Telegram.

### 1.5 Conversational Fallback (Gemini → Telegram)
If intent is chat, Gemini produces a simple response which is returned to the user.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Telegram)
**Overview:** Receives Telegram messages and starts the workflow with the message payload (chat id, text).  
**Nodes involved:** `User Sends Message to Bot`

#### Node: User Sends Message to Bot
- **Type / role:** `telegramTrigger` — entry point; listens for Telegram updates.
- **Configuration (interpreted):**
  - Subscribes to `message` updates.
  - Uses Telegram credentials (“Telegram account”).
- **Key expressions/variables:** Output provides `message.text` and `message.chat.id`.
- **Connections:**
  - **Output →** `Validate user Input`
- **Potential failures / edge cases:**
  - Telegram credential misconfiguration (webhook can’t be set / 401).
  - Bot not started by the user (privacy mode/group settings).
  - Non-text messages (stickers/photos) may not have `message.text` (expression could fail downstream).
- **Version notes:** Node typeVersion 1.2.

**Sticky note(s) applicable:**  
- “### 🕵️ Step 1: Input Classification …”  
- Also workflow-wide notes under “## ⚡ Workflow Overview & Setup …” and the YouTube embed sticky note (see tables).

---

### Block 2 — Intent Classification (OpenRouter GPT‑4.1 + Structured Output Parser)
**Overview:** Classifies the user’s message into one of two intents and returns strict JSON containing the intent type.  
**Nodes involved:** `OpenRouter Model1`, `Structured Output Parser`, `Validate user Input`, `Check For Input Type`

#### Node: OpenRouter Model1
- **Type / role:** `lmChatOpenRouter` — LLM provider connection for the classification agent.
- **Configuration:**
  - Model: `openai/gpt-4.1`
  - Credential: OpenRouter account.
- **Connections:**
  - **AI Language Model →** `Validate user Input` and `Structured Output Parser` (as provider).
- **Potential failures:**
  - OpenRouter auth/credit issues.
  - Model availability/changes.
  - Network timeouts.

#### Node: Structured Output Parser
- **Type / role:** LangChain structured output parser — enforces JSON schema for the classifier output.
- **Configuration:**
  - `autoFix: true` to attempt repairing near-JSON.
  - Example schema:
    - `{ "type": "chat" }`
- **Connections:**
  - **AI Output Parser →** `Validate user Input`
- **Edge cases:**
  - If the LLM output diverges too far, auto-fix may fail; workflow will error.
  - Schema example is minimal; it does not strictly enforce enum values.

#### Node: Validate user Input
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — intent classifier + mapper.
- **Configuration:**
  - **Input text expression:**
    - `User Input :{{ $('User Sends Message to Bot').first().json.message.text }}`
  - **System message logic:**
    - CASE 1: detect real estate/Lis Pendens/filings/records → output `{ "type": "Real_State" }`
    - CASE 2: greeting/joke/general conversation → output `{ "type": "chat" }`
  - **Output constraints:** “Return ONLY the JSON object…”
  - `hasOutputParser: true` (uses `Structured Output Parser`).
- **Outputs:** Typically becomes `{$json.output.type}` downstream (n8n LangChain agent output structure).
- **Connections:**
  - **Main →** `Check For Input Type`
- **Failure modes:**
  - If `message.text` missing, expression resolves to `undefined` and classification may be unreliable.
  - Prompt mismatch: it specifies `type` but downstream uses `$json.output.type` (correct for agent), while another node uses `$json.output.Type` (case mismatch; see chat block).

#### Node: Check For Input Type
- **Type / role:** `switch` — routes execution by intent.
- **Configuration:**
  - Rule 1: if `={{ $json.output.type }}` equals `Real_State` → Real estate branch.
  - Rule 2: if `={{ $json.output.type }}` equals `chat` → Chat branch.
- **Connections:**
  - **Output 0 (Real_State) →** `Send Alert`, `Calculate "To_Date"`, `Today's Date`
  - **Output 1 (chat) →** `Chatting With User`
- **Edge cases:**
  - If classifier outputs a different casing/spelling, no branch matches → workflow ends without response.
  - Strict type validation is enabled; non-string types could fail comparisons.

**Sticky note(s) applicable:**  
- “### 🕵️ Step 1: Input Classification …”

---

### Block 3 — Real Estate Branch: Acknowledge + Date Range Computation
**Overview:** Immediately acknowledges the user via Telegram, computes “To” and “From” dates (last 5 days), and formats them for BrowserAct input.  
**Nodes involved:** `Send Alert`, `Calculate "To_Date"`, `Today's Date`, `Calculate "From_Date"`, `Format To_Date to Specific Format`, `Format From_Date to Specific Format`, `Wait for Both Paths`

#### Node: Send Alert
- **Type / role:** `telegram` — sends an immediate acknowledgement.
- **Configuration:**
  - Text: `Ok give me few minutes.`
  - Chat ID expression: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
- **Connections:** No downstream connection (side notification).
- **Failure modes:** Telegram API errors, invalid chat id, bot blocked by user.

#### Node: Calculate "To_Date"
- **Type / role:** `dateTime` — produces a timestamp in `To_Date`.
- **Configuration:**
  - Output field: `To_Date`
  - (No explicit operation configured; defaults typically to “now”.)
- **Connections:** → `Format To_Date to Specific Format`
- **Edge cases:** Timezone defaults to instance timezone; may affect what “today” means.

#### Node: Today's Date
- **Type / role:** `dateTime` — produces a timestamp in `Today`.
- **Configuration:** Output field: `Today`
- **Connections:** → `Calculate "From_Date"`

#### Node: Calculate "From_Date"
- **Type / role:** `dateTime` — subtracts 5 days from “Today”.
- **Configuration:**
  - Operation: `addToDate`
  - Magnitude expression: `={{ $json.Today }}`
  - Duration: `-5` (days)
  - Output field: `From_Date`
- **Connections:** → `Format From_Date to Specific Format`
- **Failure modes:**
  - If `Today` missing/unexpected, the date math may error.

#### Node: Format To_Date to Specific Format
- **Type / role:** `code` — converts `To_Date` into `en-US` locale date string.
- **Configuration details:**
  - Reads: `$input.first().json.To_Date`
  - Creates: `to_date = date.toLocaleDateString('en-US')`
  - Outputs object: `{ original_date, to_date }`
- **Connections:** → `Wait for Both Paths` (input 0)
- **Edge cases:**
  - Locale output depends on runtime; typically `M/D/YYYY`. If BrowserAct expects another format, extraction may fail.

#### Node: Format From_Date to Specific Format
- **Type / role:** `code` — converts `From_Date` into `en-US` locale date string.
- **Configuration details:**
  - Reads: `$input.first().json.From_Date`
  - Creates: `from_date = date.toLocaleDateString('en-US')`
  - Outputs object: `{ original_date, from_date }`
- **Connections:** → `Wait for Both Paths` (input 1)

#### Node: Wait for Both Paths
- **Type / role:** `merge` — synchronizes both formatted dates before BrowserAct.
- **Configuration:**
  - Mode: `chooseBranch` (in practice used to wait for both incoming branches in this layout).
- **Connections:** → `Extract Lis Pendens Data`
- **Edge cases:**
  - Merge mode choice is unusual for synchronization; if only one input arrives, behavior can be surprising depending on n8n merge semantics. If the goal is strict “wait for both”, “combine”/“multiplex” patterns are often safer.

**Sticky note(s) applicable:**  
- “### 🌐 Step 2: Automated Data Extraction …” (contextually the next block, but date prep is part of the extraction pipeline).

---

### Block 4 — BrowserAct Extraction (Lis Pendens Scrape)
**Overview:** Runs a BrowserAct workflow (template) using the computed date range, returning raw results as JSON/string.  
**Nodes involved:** `Extract Lis Pendens Data`

#### Node: Extract Lis Pendens Data
- **Type / role:** `n8n-nodes-browseract.browserAct` — executes a BrowserAct cloud browser automation workflow.
- **Configuration:**
  - Run type: `WORKFLOW`
  - BrowserAct workflowId: `72087493450042645`
  - Inputs mapped:
    - `input-Today` = formatted `to_date` from `Format To_Date to Specific Format`
    - `input-From_Date` = formatted `from_date` from `Format From_Date to Specific Format`
  - Incognito: disabled
  - Credential: BrowserAct account
  - Template note indicates this corresponds to **Texas Foreclosure Leads** (per sticky note).
- **Outputs:** The next node reads `={{ $json.output.string }}` implying BrowserAct returns `output.string` containing JSON or text.
- **Connections:** → `Analyze Data & Format for Telegram`
- **Failure modes / integration issues:**
  - BrowserAct API key/workflow ID invalid.
  - Template input name mismatch (e.g., expecting different field names).
  - Scrape blocked by target website (CAPTCHA, downtime, layout changes).
  - Date format mismatch causing empty results.
  - Output not valid JSON (if downstream expects JSON array).

**Sticky note(s) applicable:**  
- “### 🌐 Step 2: Automated Data Extraction …”  
- “## ⚡ Workflow Overview & Setup …” including BrowserAct documentation links.

---

### Block 5 — AI Formatting for Telegram (OpenRouter GPT‑4.1 + Structured Output)
**Overview:** Converts scraped Lis Pendens records into Telegram-safe HTML blocks, splits into multiple messages under 3500 characters, and outputs a JSON object with `Telegram: [ ... ]`.  
**Nodes involved:** `OpenRouter Model`, `Structured Output`, `Analyze Data & Format for Telegram`

#### Node: OpenRouter Model
- **Type / role:** `lmChatOpenRouter` — LLM provider for formatting agent.
- **Configuration:**
  - Model: `openai/gpt-4.1`
  - Credentials: OpenRouter account
- **Connections:**
  - **AI Language Model →** `Analyze Data & Format for Telegram` and `Structured Output`
- **Failure modes:** Same as OpenRouter Model1.

#### Node: Structured Output
- **Type / role:** structured output parser for the formatting agent.
- **Configuration:**
  - `autoFix: true`
  - Schema example:
    - `{ "Telegram": [ "String 1 ...", "String 2 ..." ] }`
- **Connections:**
  - **AI Output Parser →** `Analyze Data & Format for Telegram`
- **Edge cases:**
  - If the LLM emits invalid JSON/escapes, autoFix may still fail.
  - Very long legal descriptions may challenge the 3500-char limit enforcement.

#### Node: Analyze Data & Format for Telegram
- **Type / role:** LangChain agent — parses raw extraction output and formats.
- **Configuration:**
  - Input text: `={{ $json.output.string }}`
  - System message includes:
    - Expected input: JSON array with keys `file_number`, `file_date`, `names`, `legal_description`, `film_code_link`
    - Output: strict JSON `{ "Telegram": [ ... ] }`
    - Telegram HTML allowed tags; names split by `;` into newline bullets; include link anchor.
    - First message includes header `📢 <b>New Lis Pendens Filings</b>\n\n`
    - Ensure each message ≤ 3500 chars and split only between records.
  - `hasOutputParser: true` (uses `Structured Output`).
- **Connections:** → `Split Out Generated Content`
- **Failure modes / edge cases:**
  - If BrowserAct output is not a JSON array (or is wrapped as a string with extra text), the agent may misparse.
  - If `film_code_link` contains characters requiring HTML escaping, Telegram may reject malformed HTML.
  - If there are zero results, prompt does not specify behavior (should it return a “no new filings” message?); without that, output may be empty and downstream splitting may behave unexpectedly.

**Sticky note(s) applicable:**  
- “### 📊 Step 3: AI Analysis & Formatting …”

---

### Block 6 — Delivery: Split Messages, Rate-limit Protection, Send to Telegram
**Overview:** Splits the `Telegram` array into individual items, iterates through them, waits between sends, and delivers HTML messages to the same chat.  
**Nodes involved:** `Split Out Generated Content`, `Loop Over Items`, `Prevent Rate Limits`, `Send Lead Data to Telegram`

#### Node: Split Out Generated Content
- **Type / role:** `splitOut` — converts an array field into multiple items.
- **Configuration:**
  - Field to split: `output.Telegram`
- **Connections:** → `Loop Over Items`
- **Edge cases:**
  - If `output.Telegram` is missing or not an array, node errors or produces no items.

#### Node: Loop Over Items
- **Type / role:** `splitInBatches` — batch iterator (used here as an item loop).
- **Configuration:** Defaults (batch size default is typically 1 unless changed).
- **Connections:**
  - Output 0 is unused.
  - Output 1 → `Prevent Rate Limits`
- **Edge cases:**
  - Mis-wired output: in n8n, SplitInBatches commonly uses **Output 0** for items and **Output 1** when done; here the connection goes from **Output 1**, which may mean messages are only sent after completion (or not at all), depending on runtime behavior/version. This is a key risk area.
  - If batch size > 1, expression used later may not match expected item shape.

#### Node: Prevent Rate Limits
- **Type / role:** `wait` — throttling between Telegram sends.
- **Configuration:** No explicit duration shown (defaults depend on node settings; could be “wait until resumed” if not set). In many cases you must set a time interval for rate limiting.
- **Connections:** → `Send Lead Data to Telegram`
- **Edge cases:**
  - If wait node is not configured with a duration, execution can stall indefinitely.

#### Node: Send Lead Data to Telegram
- **Type / role:** `telegram` — sends each formatted HTML chunk.
- **Configuration:**
  - Text: `={{ $('Loop Over Items').item.json["output.Telegram"] }}`
  - Chat ID: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
  - Parse mode: `HTML`
- **Connections:** → `Loop Over Items` (to continue iteration)
- **Failure modes:**
  - Telegram rejects HTML if malformed (`400: can't parse entities`).
  - Message too long if the 3500-char rule wasn’t respected.
  - Rate limiting (`429`) if wait is ineffective.

**Sticky note(s) applicable:**  
- “### 🚀 Step 4: Final Delivery …”

---

### Block 7 — Conversational Fallback (Gemini Chat)
**Overview:** If the user intent is chat, Gemini generates a short plain response and the bot replies in Telegram.  
**Nodes involved:** `Google Gemini`, `Chatting With User`, `Answer the User`

#### Node: Google Gemini
- **Type / role:** `lmChatGoogleGemini` — LLM provider for chat replies.
- **Configuration:**
  - Credential: “Google Gemini(PaLM) Api account”
- **Connections:**
  - **AI Language Model →** `Chatting With User`
- **Failure modes:** API key restrictions, quota, region limitations, network timeouts.

#### Node: Chatting With User
- **Type / role:** LangChain agent — generates a conversational response.
- **Configuration:**
  - Input text: `=Input type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
  - System message: if input type is chat, generate single response, raw text, no code fences.
- **Important issue:** It references `$json.output.Type` (capital `T`), but the classifier outputs `type` (lowercase) and switch checks `$json.output.type`. This likely evaluates to undefined.
- **Connections:** → `Answer the User`
- **Edge cases:** Non-text messages; undefined type; prompt may include “Input type : ” with blank value.

#### Node: Answer the User
- **Type / role:** `telegram` — sends chat response.
- **Configuration:**
  - Text: `={{ $json.output }}`
  - Chat ID: `={{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - Parse mode: `HTML` (even though system says avoid tags)
  - `appendAttribution: false`
- **Failure modes:**
  - If agent returns non-HTML but parse_mode HTML is on, it usually still works unless special characters create entity issues.
  - If `$json.output` is not a string (agent output structure differs), message may be wrong.

**Sticky note(s) applicable:**  
- “### 💬 Step 2-2: Conversational Fallback …”

---

### Block 8 — Documentation / Annotations (Sticky Notes)
**Overview:** Non-executing documentation nodes describing setup and steps, including BrowserAct links and a YouTube embed reference.  
**Nodes involved:** `Documentation`, `Step 1 Explanation`, `Step 2 Explanation`, `Step 3 Explanation`, `Step 4 Explanation`, `Step 4 Explanation1`, `Sticky Note`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Workflow entry point (Telegram inbound message) | — | Validate user Input | ### 🕵️ Step 1: Input Classification<br>The workflow triggers via Telegram and uses an AI agent to distinguish between a casual chat and a specific real estate records request. If a data request is detected, the system immediately calculates a dynamic 5-day date range for the search.<br><br>## ⚡ Workflow Overview & Setup<br>**Summary:** This automation monitors real estate public records (specifically Lis Pendens filings) via Telegram, using BrowserAct to scrape government databases and Gemini to deliver structured lead reports.<br>…<br>[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)<br>[How to Connect n8n to BrowserAct](https://docs.browseract.com)<br>[How to Use & Customize BrowserAct Templates](https://docs.browseract.com)<br><br>@[youtube](Q2zUXDDhD8w) |
| Validate user Input | @n8n/n8n-nodes-langchain.agent | Classify intent and output strict JSON | User Sends Message to Bot | Check For Input Type | ### 🕵️ Step 1: Input Classification … |
| OpenRouter Model1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider for intent classifier | — | (AI provider to Validate user Input / Structured Output Parser) | ### 🕵️ Step 1: Input Classification … |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON for classifier | — | (AI parser to Validate user Input) | ### 🕵️ Step 1: Input Classification … |
| Check For Input Type | switch | Route to Real_State vs chat branch | Validate user Input | Send Alert; Calculate "To_Date"; Today's Date; Chatting With User | ### 🕵️ Step 1: Input Classification … |
| Send Alert | telegram | Acknowledge request immediately | Check For Input Type | — | ### 🌐 Step 2: Automated Data Extraction … |
| Calculate "To_Date" | dateTime | Generate To_Date (now) | Check For Input Type | Format To_Date to Specific Format | ### 🌐 Step 2: Automated Data Extraction … |
| Today's Date | dateTime | Generate Today timestamp | Check For Input Type | Calculate "From_Date" | ### 🌐 Step 2: Automated Data Extraction … |
| Calculate "From_Date" | dateTime | Compute From_Date = Today - 5 days | Today's Date | Format From_Date to Specific Format | ### 🌐 Step 2: Automated Data Extraction … |
| Format To_Date to Specific Format | code | Convert To_Date to en-US date string | Calculate "To_Date" | Wait for Both Paths | ### 🌐 Step 2: Automated Data Extraction … |
| Format From_Date to Specific Format | code | Convert From_Date to en-US date string | Calculate "From_Date" | Wait for Both Paths | ### 🌐 Step 2: Automated Data Extraction … |
| Wait for Both Paths | merge | Synchronize both date outputs | Format To_Date…; Format From_Date… | Extract Lis Pendens Data | ### 🌐 Step 2: Automated Data Extraction … |
| Extract Lis Pendens Data | browserAct | Run BrowserAct workflow scrape using date inputs | Wait for Both Paths | Analyze Data & Format for Telegram | ### 🌐 Step 2: Automated Data Extraction …<br><br>## ⚡ Workflow Overview & Setup … (BrowserAct template requirement + links) |
| OpenRouter Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider for Telegram formatting | — | (AI provider to Analyze Data… / Structured Output) | ### 📊 Step 3: AI Analysis & Formatting … |
| Structured Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce `{Telegram:[...]}` JSON | — | (AI parser to Analyze Data & Format for Telegram) | ### 📊 Step 3: AI Analysis & Formatting … |
| Analyze Data & Format for Telegram | @n8n/n8n-nodes-langchain.agent | Convert raw filings to Telegram HTML chunks | Extract Lis Pendens Data | Split Out Generated Content | ### 📊 Step 3: AI Analysis & Formatting … |
| Split Out Generated Content | splitOut | Turn `output.Telegram[]` into separate items | Analyze Data & Format for Telegram | Loop Over Items | ### 🚀 Step 4: Final Delivery … |
| Loop Over Items | splitInBatches | Iterate messages (batch loop) | Split Out Generated Content; Send Lead Data to Telegram | Prevent Rate Limits | ### 🚀 Step 4: Final Delivery … |
| Prevent Rate Limits | wait | Throttle between Telegram sends | Loop Over Items | Send Lead Data to Telegram | ### 🚀 Step 4: Final Delivery … |
| Send Lead Data to Telegram | telegram | Send each HTML message chunk | Prevent Rate Limits | Loop Over Items | ### 🚀 Step 4: Final Delivery … |
| Google Gemini | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for chat fallback | — | (AI provider to Chatting With User) | ### 💬 Step 2-2: Conversational Fallback<br>This branch engages the user in natural conversation and answers them. |
| Chatting With User | @n8n/n8n-nodes-langchain.agent | Generate chat response | Check For Input Type | Answer the User | ### 💬 Step 2-2: Conversational Fallback … |
| Answer the User | telegram | Send chat response to Telegram | Chatting With User | — | ### 💬 Step 2-2: Conversational Fallback … |
| Documentation | stickyNote | Setup/requirements notes | — | — | ## ⚡ Workflow Overview & Setup<br>…<br>[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)<br>[How to Connect n8n to BrowserAct](https://docs.browseract.com)<br>[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | stickyNote | Block explanation | — | — | ### 🕵️ Step 1: Input Classification … |
| Step 2 Explanation | stickyNote | Block explanation | — | — | ### 🌐 Step 2: Automated Data Extraction … |
| Step 3 Explanation | stickyNote | Block explanation | — | — | ### 📊 Step 3: AI Analysis & Formatting … |
| Step 4 Explanation | stickyNote | Block explanation | — | — | ### 🚀 Step 4: Final Delivery … |
| Step 4 Explanation1 | stickyNote | Block explanation | — | — | ### 💬 Step 2-2: Conversational Fallback … |
| Sticky Note | stickyNote | YouTube embed reference | — | — | @[youtube](Q2zUXDDhD8w) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Telegram Trigger** (`telegramTrigger`)
   2. Updates: **message**
   3. Configure **Telegram API credentials** (Bot token via n8n Telegram credential).

2) **Add Intent Classification (OpenRouter + structured JSON)**
   1. Add node: **OpenRouter Chat Model** (`lmChatOpenRouter`)
      - Model: `openai/gpt-4.1`
      - Configure **OpenRouter API credential**
   2. Add node: **Structured Output Parser** (`outputParserStructured`)
      - Enable **Auto-fix**
      - Schema example: `{ "type": "chat" }`
   3. Add node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`) named “Validate user Input”
      - Prompt type: **Define**
      - Text field expression: `User Input :{{ $('User Sends Message to Bot').first().json.message.text }}`
      - System message: paste the classifier rules (Real_State vs chat) and require strict JSON only
      - Enable “Has Output Parser” and attach the **Structured Output Parser**
      - Attach **OpenRouter model** as Language Model
   4. Connect: **Telegram Trigger → Validate user Input**

3) **Route by intent**
   1. Add node: **Switch**
      - Rule A: `{{ $json.output.type }}` equals `Real_State`
      - Rule B: `{{ $json.output.type }}` equals `chat`
   2. Connect: **Validate user Input → Switch**

4) **Real_State branch: acknowledge + compute dates**
   1. Add node: **Telegram** (send message) named “Send Alert”
      - Chat ID: `{{ $('User Sends Message to Bot').first().json.message.chat.id }}`
      - Text: `Ok give me few minutes.`
   2. Add node: **Date & Time** named “Calculate "To_Date"”
      - Output field: `To_Date` (leave default = now)
   3. Add node: **Date & Time** named “Today's Date”
      - Output field: `Today`
   4. Add node: **Date & Time** named “Calculate "From_Date"”
      - Operation: addToDate
      - Base/magnitude: `{{ $json.Today }}`
      - Duration: `-5` days
      - Output field: `From_Date`
   5. Add node: **Code** named “Format To_Date to Specific Format”
      - Convert `To_Date` to `toLocaleDateString('en-US')`, output `{to_date}`
   6. Add node: **Code** named “Format From_Date to Specific Format”
      - Convert `From_Date` to `toLocaleDateString('en-US')`, output `{from_date}`
   7. Add node: **Merge** named “Wait for Both Paths”
      - Use a mode that reliably waits/combines both inputs (the original uses `chooseBranch`; consider using a merge mode that combines inputs deterministically).
   8. Wire Real_State output from Switch to:
      - **Send Alert**
      - **Calculate "To_Date" → Format To_Date → Merge input 0**
      - **Today's Date → Calculate "From_Date" → Format From_Date → Merge input 1**

5) **BrowserAct extraction**
   1. Add node: **BrowserAct**
      - Type: WORKFLOW
      - Workflow ID: `72087493450042645`
      - Map inputs:
        - `input-Today` = `{{ $('Format To_Date to Specific Format').first().json.to_date }}`
        - `input-From_Date` = `{{ $('Format From_Date to Specific Format').first().json.from_date }}`
      - Configure **BrowserAct API credential**
      - Ensure the BrowserAct account contains the referenced template/workflow (notably “Texas Foreclosure Leads”).
   2. Connect: **Merge → BrowserAct**

6) **Format scraped data for Telegram (OpenRouter + structured JSON)**
   1. Add node: **OpenRouter Chat Model** (second instance) model `openai/gpt-4.1`
   2. Add node: **Structured Output Parser** with schema example:
      - `{ "Telegram": ["String 1", "String 2"] }`
   3. Add node: **AI Agent** named “Analyze Data & Format for Telegram”
      - Text: `{{ $json.output.string }}`
      - System message: paste the Telegram formatting rules (HTML tags allowed, header, 3500 char split, etc.)
      - Enable “Has Output Parser” and connect the structured output parser
      - Attach the OpenRouter model
   4. Connect: **BrowserAct → Analyze Data & Format for Telegram**

7) **Split messages and send with throttling**
   1. Add node: **Split Out** (`splitOut`)
      - Field: `output.Telegram`
   2. Add node: **Split In Batches** named “Loop Over Items”
      - Batch size: typically `1` for sequential send
      - Ensure you connect from the correct output (commonly Output 0 processes items)
   3. Add node: **Wait** named “Prevent Rate Limits”
      - Configure a delay (e.g., 1–2 seconds) appropriate for Telegram rate limits
   4. Add node: **Telegram** named “Send Lead Data to Telegram”
      - Text: `{{ $json["output.Telegram"] }}` (or from the current item)
      - Chat ID: `{{ $('User Sends Message to Bot').first().json.message.chat.id }}`
      - Parse mode: `HTML`
   5. Connect:
      - **Analyze Data & Format → Split Out → Split In Batches → Wait → Telegram Send**
      - Loop back from **Telegram Send → Split In Batches** to continue batches (standard batching pattern).

8) **Chat branch (Gemini)**
   1. Add node: **Google Gemini Chat Model** (`lmChatGoogleGemini`)
      - Configure Gemini/PaLM credentials
   2. Add node: **AI Agent** named “Chatting With User”
      - Text: include the user message; use the correct field name from classifier: `{{ $json.output.type }}`
      - System message: produce a single raw text response, no code fences
      - Attach Gemini as language model
   3. Add node: **Telegram** named “Answer the User”
      - Text: use the agent’s output string (confirm actual agent output path in your n8n version)
      - Chat ID: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
   4. Connect: **Switch(chat) → Chatting With User → Answer the User**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Requirements: Telegram, BrowserAct, OpenRouter (GPT-4), Google Gemini (PaLM). Mandatory: BrowserAct API + template “Texas Foreclosure Leads”. | From sticky note “## ⚡ Workflow Overview & Setup” |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| YouTube embed reference: `@[youtube](Q2zUXDDhD8w)` | From sticky note “@[youtube](Q2zUXDDhD8w)” |

