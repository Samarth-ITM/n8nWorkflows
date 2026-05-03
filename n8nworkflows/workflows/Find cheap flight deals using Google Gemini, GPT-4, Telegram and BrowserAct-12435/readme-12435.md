Find cheap flight deals using Google Gemini, GPT-4, Telegram and BrowserAct

https://n8nworkflows.xyz/workflows/find-cheap-flight-deals-using-google-gemini--gpt-4--telegram-and-browseract-12435


# Find cheap flight deals using Google Gemini, GPT-4, Telegram and BrowserAct

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Find cheap flight deals using AI, Telegram and BrowserAct  
**Purpose:** A Telegram travel bot that classifies user intent (travel vs chat vs missing info), extracts the **departure city**, triggers a **BrowserAct** automation to scrape low-cost flight deals, then uses **GPT‑4.1 (OpenRouter)** to curate results into **Telegram-ready HTML** messages (split to respect message length), and sends them back to the user.

**Typical use cases**
- User: “Find cheap flights from Berlin” → bot scrapes deals and returns a formatted list.
- User: “I want cheap flights” → bot asks for departure city.
- User: “Hi / what can you do?” → bot responds conversationally.

### 1.1 Input Reception (Telegram)
Receives Telegram messages and forwards text into the AI intent classifier.

### 1.2 Intent Classification & Entity Extraction (Gemini + Structured Parsing)
Google Gemini is used as the language model for an agent that outputs strict JSON (`type`, `location`). A structured output parser validates/corrects the JSON.

### 1.3 Branching by Intent
A Switch routes execution:
- `travel` → notify user + run BrowserAct scraping workflow
- `chat` / `nodata` → conversational fallback (answer or ask for departure city)

### 1.4 Deal Scraping (BrowserAct Sub-workflow)
Runs a BrowserAct workflow (“Low-Const Travel Finder”) with `input-Location` to retrieve raw flight deal data.

### 1.5 AI Curation & Telegram Delivery (GPT‑4.1 + Structured Parsing)
GPT‑4.1 formats/sorts the scraped flight data into one or more HTML Telegram messages (JSON array), splits messages, loops over them, waits to avoid rate limits, and sends sequentially.

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram Intake
**Overview:** Starts the workflow when a Telegram message is received and provides the message payload to downstream nodes.  
**Nodes involved:** `User Sends Message to Bot`

#### Node: User Sends Message to Bot
- **Type / role:** `telegramTrigger` — entry point; listens for Telegram updates.
- **Configuration (interpreted):**
  - Listens to update type: **message**
  - Uses Telegram bot credentials (`Telegram account`)
- **Key data used downstream:**
  - `$json.message.text` (user message text)
  - `$json.message.chat.id` (chat to reply to)
- **Connections:**
  - **Output →** `Validate user Input`
- **Version requirements:** node typeVersion `1.2` (ensure your n8n has compatible Telegram Trigger version).
- **Edge cases / failures:**
  - Bot not reachable / invalid webhook / polling issues (depends on Telegram setup in n8n)
  - Missing `message.text` for non-text messages (stickers, photos). This workflow assumes text; non-text updates may cause expression errors in the classifier.

**Sticky note(s) covering this block:**
- “### 🕵️ Step 1: Intent Analysis …”
- “@[youtube](W4EYza-Te5w)” (general video reference)
- “## ⚡ Workflow Overview & Setup …” (requirements & links)

---

### Block 2 — Intent Classification & Structured Output
**Overview:** Uses an LLM (Gemini) to classify the message as `travel`, `nodata`, or `chat`, and extracts `location` when present. Output is forced into strict JSON via a structured output parser.  
**Nodes involved:** `Validate user Inputs` (Gemini model), `Structured Output Parser`, `Validate user Input` (agent)

#### Node: Validate user Inputs
- **Type / role:** `lmChatGoogleGemini` — provides a Gemini chat model to the agent.
- **Configuration:**
  - Uses Google Gemini (PaLM) credentials: `Google Gemini(PaLM) Api account`
  - Default options (no custom params shown)
- **Connections:**
  - **AI languageModel →** `Validate user Input`
  - **AI languageModel →** `Structured Output Parser` (used to auto-fix/validate structured outputs)
- **Edge cases / failures:**
  - Invalid/expired API key, quota limits
  - Model response latency/timeouts

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — validates and auto-fixes the JSON schema for the classifier.
- **Configuration:**
  - `autoFix: true` (attempt to correct malformed JSON)
  - Schema example enforced conceptually:
    ```json
    { "type": "travel", "location": "San Francisco" }
    ```
- **Connections:**
  - **AI outputParser →** `Validate user Input`
- **Edge cases / failures:**
  - If the model returns text that cannot be repaired into valid JSON
  - If returned keys differ (e.g., `Type` vs `type`) the downstream Switch may not match

#### Node: Validate user Input
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — intent classifier + entity extractor.
- **Configuration:**
  - **Input text:** `={{ $json.message.text }}`
  - **System message:** strict instructions to output **only raw JSON**:
    - `type`: `travel` | `nodata` | `chat`
    - `location`: city name or `null`
  - **Output parser enabled:** yes (`hasOutputParser: true`)
- **Connections:**
  - **Main output →** `Check For Input Type`
  - Uses:
    - **Language model:** `Validate user Inputs` (Gemini)
    - **Output parser:** `Structured Output Parser`
- **Edge cases / failures:**
  - If Telegram message is missing `.text`, the expression fails
  - Misclassification (e.g., user provides destination but no departure; prompt only extracts departure)
  - City ambiguity (“Paris” TX vs FR) not resolved in prompt

**Sticky note(s) covering this block:**
- “### 🕵️ Step 1: Intent Analysis …”

---

### Block 3 — Branching: Travel vs Chat vs Missing Data
**Overview:** Routes execution based on `output.type` returned by the classifier.  
**Nodes involved:** `Check For Input Type`

#### Node: Check For Input Type
- **Type / role:** `switch` — conditional branching.
- **Configuration:**
  - Evaluates `={{ $json.output.type }}`
  - Rules:
    1. equals `travel` → Travel branch
    2. equals `chat` → Chat branch
    3. equals `nodata` → Missing-data branch
- **Connections:**
  - **Travel output (rule 1) →** `Process Initialization Alert` and `Run "Low-Const Travel Finder" workflow` (two parallel outputs from same branch)
  - **Chat output (rule 2) →** `Chatting With User`
  - **No-data output (rule 3) →** `Chatting With User`
- **Edge cases / failures:**
  - If classifier returns unexpected casing/field name, no rule matches → nothing happens
  - If the agent returned `output.Type` instead of `output.type` (note: elsewhere the workflow references `output.Type`, see Block 4) this causes inconsistencies

**Sticky note(s):**
- “### 🕵️ Step 1: Intent Analysis …”

---

### Block 4 — Conversational Fallback (Chat or Missing Departure City)
**Overview:** Generates a direct reply when the user is chatting, or asks for the departure location when `nodata`. Sends the response to Telegram.  
**Nodes involved:** `Google Gemini`, `Chatting With User`, `Answer the User`

#### Node: Google Gemini
- **Type / role:** `lmChatGoogleGemini` — LLM provider for the conversational agent.
- **Configuration:**
  - Uses the same Gemini credentials as earlier
- **Connections:**
  - **AI languageModel →** `Chatting With User`
- **Edge cases / failures:**
  - API quota/auth issues
  - If you expect consistent formatting, Gemini may vary unless heavily constrained

#### Node: Chatting With User
- **Type / role:** `langchain.agent` — generates a raw text response.
- **Configuration:**
  - **Text input expression:**
    ```
    =Input type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}
    ```
    **Important:** This references `output.Type` (capital T), but the classifier specification outputs `type` lowercase. This is likely a bug; it may produce “undefined” input type.
  - **System message:**
    - If `nodata`: ask for departure location
    - If `chat`: respond conversationally
    - Output must be raw text (no code fences)
- **Connections:**
  - **Main output →** `Answer the User`
  - Uses **language model:** `Google Gemini`
- **Edge cases / failures:**
  - The `output.Type` vs `output.type` mismatch can reduce response quality or logic.
  - For `nodata`, the agent is expected to ask for departure city, but it does not enforce a structured format.

#### Node: Answer the User
- **Type / role:** `telegram` — sends a Telegram message back to the user.
- **Configuration:**
  - **Text:** `={{ $json.output }}`
  - **Chat ID:** `={{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - Parse mode: `HTML` (though the conversational agent outputs plain text; HTML is still acceptable if it doesn’t contain unsupported tags)
  - Attribution disabled
- **Connections:** terminal for chat/nodata path
- **Edge cases / failures:**
  - Telegram HTML parse errors if the model emits invalid HTML entities/tags
  - Chat ID expression fails if the trigger item is not present as expected

**Sticky note(s):**
- “### 💬 Step 2-2: Conversational Fallback …” (note: content mentions “URL”, but the actual workflow logic is about *departure location*, not URL)

---

### Block 5 — Travel Branch: Notify User + Run BrowserAct Scrape
**Overview:** For `travel` intent, immediately tells the user to wait, then launches a BrowserAct workflow to scrape flight deals using the extracted departure location.  
**Nodes involved:** `Process Initialization Alert`, `Run "Low-Const Travel Finder" workflow`

#### Node: Process Initialization Alert
- **Type / role:** `telegram` — sends a quick acknowledgement.
- **Configuration:**
  - Text: `Ok give me few minutes.`
  - Chat ID: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
- **Connections:** no downstream (pure notification)
- **Edge cases / failures:**
  - Telegram send failures due to rate limits or auth
  - If multiple items exist, `.first()` ensures first item chat ID is used (fine for trigger-based flow)

#### Node: Run "Low-Const Travel Finder" workflow
- **Type / role:** `browserAct` — runs a BrowserAct managed workflow (“WORKFLOW” type).
- **Configuration:**
  - Mode: `WORKFLOW`
  - Timeout: `7200` seconds (2 hours)
  - Workflow ID: `72181632540107285`
  - Workflow inputs mapping:
    - `input-Location` = `={{ $json.output.location }}`
    - `input-Momondo` exists in schema but is marked removed (not passed)
  - Incognito: false
  - Credentials: `BrowserAct account`
- **Connections:**
  - **Main output →** `Analyze Data & Generate Response`
- **Edge cases / failures:**
  - BrowserAct workflow ID not found / permission denied
  - Workflow execution timeout (especially if site is slow or blocked)
  - Scrape output shape mismatch: downstream expects `output.string` to exist
  - Anti-bot measures / CAPTCHAs on travel sites

**Sticky note(s):**
- “### ✈️ Step 2: Automated Deal Scrape …”
- Documentation sticky note includes BrowserAct links.

---

### Block 6 — AI Curation (GPT‑4.1 via OpenRouter) + Structured Telegram Output
**Overview:** Takes raw scraped flight data and departure city, asks GPT‑4.1 to sort/group/format as HTML, enforcing a JSON output schema `{ Telegram: [ ... ] }`.  
**Nodes involved:** `OpenRouter Model`, `Structured Output`, `Analyze Data & Generate Response`

#### Node: OpenRouter Model
- **Type / role:** `lmChatOpenRouter` — provides GPT‑4.1 via OpenRouter for formatting.
- **Configuration:**
  - Model: `openai/gpt-4.1`
  - Credentials: `OpenRouter account`
- **Connections:**
  - **AI languageModel →** `Analyze Data & Generate Response`
  - **AI languageModel →** `Structured Output` (parser uses same model connection structure in n8n LangChain integration)
- **Edge cases / failures:**
  - OpenRouter auth/credit limits
  - Model availability changes
  - Higher latency/cost for large scraped payloads

#### Node: Structured Output
- **Type / role:** `outputParserStructured` — ensures final response is valid JSON with Telegram message array.
- **Configuration:**
  - `autoFix: true`
  - Schema example:
    ```json
    { "Telegram": ["String 1 ...", "String 2 ..."] }
    ```
- **Connections:**
  - **AI outputParser →** `Analyze Data & Generate Response`
- **Edge cases / failures:**
  - If the model outputs non-JSON or unfixable JSON
  - If messages exceed Telegram limits despite instructions

#### Node: Analyze Data & Generate Response
- **Type / role:** `langchain.agent` — transforms raw flight data into curated Telegram HTML messages.
- **Configuration:**
  - **Text input:**
    - `Travels Data :{{ $json.output.string }},`
    - `Departure Location : {{ $('Validate user Input').first().json.output.location }}`
  - **System message:** detailed formatter/curator instructions:
    - Sort by price ascending
    - Group if long
    - Use allowed Telegram HTML tags
    - Infer flags by city
    - Split if >3500 chars into multiple array elements
    - Output **only JSON** with `Telegram` array
  - **Output parser enabled:** yes (`hasOutputParser: true`) using `Structured Output`
  - **Language model:** OpenRouter GPT‑4.1
- **Connections:**
  - **Main output →** `Split Out Generated Data`
- **Edge cases / failures:**
  - If BrowserAct output isn’t present at `$json.output.string`
  - Flag inference may be wrong (city duplicates across countries)
  - If scraped data isn’t valid JSON array as described, the agent may hallucinate structure
  - HTML escaping issues can cause Telegram parse errors

**Sticky note(s):**
- “### 📊 Step 3: AI Curation & Delivery …”

---

### Block 7 — Split, Rate Limit Protection, and Sequential Telegram Sends
**Overview:** Splits the `Telegram` array into items, iterates over them, waits to reduce Telegram rate limit risk, then sends each message.  
**Nodes involved:** `Split Out Generated Data`, `Loop Over Items`, `Avoid Rate Limits`, `Send Travel List to User`

#### Node: Split Out Generated Data
- **Type / role:** `splitOut` — converts an array field into individual items.
- **Configuration:**
  - Field to split: `output.Telegram`
- **Connections:**
  - **Main output →** `Loop Over Items`
- **Edge cases / failures:**
  - If `output.Telegram` is missing or not an array → node errors

#### Node: Loop Over Items
- **Type / role:** `splitInBatches` — batch iteration controller.
- **Configuration:**
  - Default batch settings (not specified)
- **Connections:**
  - **Output (second/main “continue” path in this workflow’s wiring) →** `Avoid Rate Limits`
  - Also receives input from `Send Travel List to User` to continue looping (see below)
- **Edge cases / failures:**
  - Mis-wired loop logic can cause infinite loops or skipping (here it forms a send→loop pattern)
  - Batch size defaults may affect pacing

#### Node: Avoid Rate Limits
- **Type / role:** `wait` — delay between messages.
- **Configuration:** no explicit duration shown (defaults depend on node configuration; in many cases it requires a set time—verify in UI).
- **Connections:**
  - **Main output →** `Send Travel List to User`
- **Edge cases / failures:**
  - If configured as “wait for webhook” unintentionally, execution may stall
  - If delay is too short, Telegram may still rate-limit

#### Node: Send Travel List to User
- **Type / role:** `telegram` — sends one curated HTML chunk.
- **Configuration:**
  - Text: `={{ $('Loop Over Items').item.json["output.Telegram"] }}`
  - Chat ID: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
  - Parse mode: HTML
- **Connections:**
  - **Main output →** `Loop Over Items` (forms the iterative cycle)
- **Edge cases / failures:**
  - Telegram HTML parsing errors if message contains unsupported/invalid HTML
  - If message exceeds Telegram max length, send fails (even though agent tries to cap at ~3500 chars)
  - If the loop state is incorrect, it may resend same item or stop early

**Sticky note(s):**
- “### 📊 Step 3: AI Curation & Delivery …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | Telegram Trigger | Entry point: receive Telegram messages | — | Validate user Input | ### 🕵️ Step 1: Intent Analysis … |
| Validate user Input | LangChain Agent | Classify intent + extract departure city into JSON | User Sends Message to Bot | Check For Input Type | ### 🕵️ Step 1: Intent Analysis … |
| Validate user Inputs | Google Gemini Chat Model | LLM backend for intent classifier | — | (AI) Validate user Input; (AI) Structured Output Parser | ### 🕵️ Step 1: Intent Analysis … |
| Structured Output Parser | Structured Output Parser | Enforce `{type, location}` JSON | — | (AI) Validate user Input | ### 🕵️ Step 1: Intent Analysis … |
| Check For Input Type | Switch | Route by `output.type` (travel/chat/nodata) | Validate user Input | (travel) Process Initialization Alert + Run "Low-Const Travel Finder" workflow; (chat/nodata) Chatting With User | ### 🕵️ Step 1: Intent Analysis … |
| Chatting With User | LangChain Agent | Chat response or ask for missing departure city | Check For Input Type | Answer the User | ### 💬 Step 2-2: Conversational Fallback … |
| Google Gemini | Google Gemini Chat Model | LLM backend for conversational agent | — | (AI) Chatting With User | ### 💬 Step 2-2: Conversational Fallback … |
| Answer the User | Telegram | Send fallback reply to user | Chatting With User | — | ### 💬 Step 2-2: Conversational Fallback … |
| Process Initialization Alert | Telegram | Notify user the scrape is running | Check For Input Type (travel) | — | ### ✈️ Step 2: Automated Deal Scrape … |
| Run "Low-Const Travel Finder" workflow | BrowserAct | Run BrowserAct scraping workflow with location input | Check For Input Type (travel) | Analyze Data & Generate Response | ### ✈️ Step 2: Automated Deal Scrape … |
| Analyze Data & Generate Response | LangChain Agent | Curate/sort/group deals into Telegram HTML chunks JSON | Run "Low-Const Travel Finder" workflow | Split Out Generated Data | ### 📊 Step 3: AI Curation & Delivery … |
| OpenRouter Model | OpenRouter Chat Model | GPT‑4.1 backend for curation agent | — | (AI) Analyze Data & Generate Response; (AI) Structured Output | ### 📊 Step 3: AI Curation & Delivery … |
| Structured Output | Structured Output Parser | Enforce `{Telegram: [ ... ]}` JSON | — | (AI) Analyze Data & Generate Response | ### 📊 Step 3: AI Curation & Delivery … |
| Split Out Generated Data | Split Out | Split `output.Telegram[]` into items | Analyze Data & Generate Response | Loop Over Items | ### 📊 Step 3: AI Curation & Delivery … |
| Loop Over Items | Split In Batches | Iterate over message chunks | Split Out Generated Data; Send Travel List to User | Avoid Rate Limits | ### 📊 Step 3: AI Curation & Delivery … |
| Avoid Rate Limits | Wait | Delay between Telegram sends | Loop Over Items | Send Travel List to User | ### 📊 Step 3: AI Curation & Delivery … |
| Send Travel List to User | Telegram | Send one HTML chunk, then continue loop | Avoid Rate Limits | Loop Over Items | ### 📊 Step 3: AI Curation & Delivery … |
| Step 1 Explanation | Sticky Note | Comment | — | — | ### 🕵️ Step 1: Intent Analysis … |
| Step 2 Explanation | Sticky Note | Comment | — | — | ### ✈️ Step 2: Automated Deal Scrape … |
| Step 3 Explanation | Sticky Note | Comment | — | — | ### 📊 Step 3: AI Curation & Delivery … |
| Step 4 Explanation | Sticky Note | Comment | — | — | ### 💬 Step 2-2: Conversational Fallback … |
| Documentation | Sticky Note | Comment / requirements / links | — | — | ## ⚡ Workflow Overview & Setup … |
| Sticky Note | Sticky Note | Video reference | — | — | @[youtube](W4EYza-Te5w) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Find cheap flight deals using AI, Telegram and BrowserAct*.

2. **Add Telegram Trigger**
   - Node: **Telegram Trigger** (“User Sends Message to Bot”)
   - Updates: `message`
   - Credentials: create/select **Telegram API** credentials for your bot.

3. **Add Gemini model for intent parsing**
   - Node: **Google Gemini Chat Model** (“Validate user Inputs”)
   - Credentials: **Google Gemini (PaLM) API** key/account.

4. **Add Structured Output Parser for intent**
   - Node: **Structured Output Parser** (“Structured Output Parser”)
   - Enable **Auto-fix**
   - Provide a schema example like:
     - `type`: travel/nodata/chat
     - `location`: string or null

5. **Add Intent Classifier Agent**
   - Node: **AI Agent (LangChain Agent)** (“Validate user Input”)
   - Text: `{{ $json.message.text }}`
   - System message: use the classifier spec (travel/nodata/chat) and enforce raw JSON only.
   - Attach:
     - **Language Model:** connect from “Validate user Inputs” to the agent’s **ai_languageModel**
     - **Output Parser:** connect from “Structured Output Parser” to the agent’s **ai_outputParser**

6. **Add Switch**
   - Node: **Switch** (“Check For Input Type”)
   - Create 3 rules checking `{{ $json.output.type }}`:
     - equals `travel`
     - equals `chat`
     - equals `nodata`
   - Connect “Validate user Input” → “Check For Input Type”.

7. **Chat/Nodata branch (fallback)**
   1. Add **Google Gemini Chat Model** (“Google Gemini”) with same Gemini credentials.
   2. Add **AI Agent** (“Chatting With User”)
      - Text: include input type and user message.
      - **Fix recommended:** use `{{ $json.output.type }}` (lowercase) instead of `output.Type`.
      - System message: if `nodata` ask for departure city; if `chat` respond normally; output raw text only.
      - Connect “Google Gemini” (ai_languageModel) → “Chatting With User”.
   3. Add **Telegram** node (“Answer the User”)
      - Chat ID: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
      - Text: `{{ $json.output }}`
      - Parse mode: HTML (optional; can be plain)
   4. Connect Switch outputs for `chat` and `nodata` → “Chatting With User” → “Answer the User”.

8. **Travel branch: notify + BrowserAct**
   1. Add **Telegram** node (“Process Initialization Alert”)
      - Chat ID: `{{ $('User Sends Message to Bot').first().json.message.chat.id }}`
      - Text: “Ok give me few minutes.”
   2. Add **BrowserAct** node (“Run "Low-Const Travel Finder" workflow”)
      - Type: `WORKFLOW`
      - Workflow ID: your BrowserAct workflow ID (here: `72181632540107285`)
      - Timeout: `7200`
      - Map input:
        - `input-Location` = `{{ $json.output.location }}`
      - Credentials: **BrowserAct API** key/account.
   3. Connect Switch `travel` output → both “Process Initialization Alert” and BrowserAct node.

9. **Add OpenRouter GPT‑4.1 model**
   - Node: **OpenRouter Chat Model** (“OpenRouter Model”)
   - Model: `openai/gpt-4.1`
   - Credentials: OpenRouter API key.

10. **Add Structured Output Parser for Telegram messages**
   - Node: **Structured Output Parser** (“Structured Output”)
   - Auto-fix on
   - Schema example:
     - `{ "Telegram": ["...HTML...", "..."] }`

11. **Add Curation Agent**
   - Node: **AI Agent** (“Analyze Data & Generate Response”)
   - Text: include:
     - scraped flight data (from BrowserAct output)
     - departure location from classifier
   - System message: formatter spec (sort by price, group, HTML tags, split at 3500 chars, output JSON only).
   - Connect:
     - OpenRouter Model → agent **ai_languageModel**
     - Structured Output → agent **ai_outputParser**
   - Connect BrowserAct → curation agent.

12. **Split array into separate items**
   - Node: **Split Out** (“Split Out Generated Data”)
   - Field: `output.Telegram`
   - Connect curation agent → Split Out.

13. **Loop and send chunks**
   1. Node: **Split In Batches** (“Loop Over Items”) — default settings are acceptable initially.
   2. Node: **Wait** (“Avoid Rate Limits”) — configure a small delay (e.g., 1–2 seconds) to reduce Telegram rate limiting.
   3. Node: **Telegram** (“Send Travel List to User”)
      - Chat ID: `{{ $('User Sends Message to Bot').first().json.message.chat.id }}`
      - Text: `{{ $('Loop Over Items').item.json["output.Telegram"] }}`
      - Parse mode: HTML
   4. Wire:
      - Split Out → Loop Over Items
      - Loop Over Items → Wait → Telegram Send
      - Telegram Send → Loop Over Items (to continue batches)

14. **Activate and test**
   - Send messages:
     - “Find me cheap flights from Berlin”
     - “I want cheap flights”
     - “Hello”
   - Verify the BrowserAct workflow returns the expected field used by the curation agent (the current workflow expects something like `$json.output.string`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Requirements:** Telegram, BrowserAct, OpenRouter (GPT‑4), Google Gemini (PaLM). **Mandatory:** BrowserAct API (Template: **Low-Cost Travel Finder**) | From “Documentation” sticky note |
| “Send a message like ‘Low-Cost Travel Finder For London’ to your Telegram bot…” | From “Documentation” sticky note |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | `@[youtube](W4EYza-Te5w)` |

### Notable implementation issues to address when modifying
- **Bug risk:** `Chatting With User` uses `{{ $json.output.Type }}` but the classifier outputs `type`. Change to `{{ $json.output.type }}` for consistent behavior.
- **Wait node configuration:** ensure it is a time-based delay (not “wait for webhook”), otherwise the sending loop may stall.
- **Non-text Telegram updates:** consider filtering out messages without `message.text` to avoid expression errors.