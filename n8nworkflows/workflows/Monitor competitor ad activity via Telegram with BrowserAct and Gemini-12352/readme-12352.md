Monitor competitor ad activity via Telegram with BrowserAct and Gemini

https://n8nworkflows.xyz/workflows/monitor-competitor-ad-activity-via-telegram-with-browseract-and-gemini-12352


# Monitor competitor ad activity via Telegram with BrowserAct and Gemini

## 1. Workflow Overview

**Purpose:**  
This workflow turns a Telegram bot into a competitor ad-intelligence assistant. A user messages a brand/company name in natural language, the workflow extracts the company name with Gemini, launches a BrowserAct scraping workflow to collect Meta/Google ad activity, then uses Gemini again to summarize findings and return an HTML-formatted “strategic verdict” back to Telegram.

**Target use cases:**
- Quick competitive checks (“Are they running ads right now?”)
- Lightweight media-buying decision support (“Advertise now vs wait/monitor”)
- Automated ad-spy reporting delivered in chat

### 1.1 Input Reception (Telegram)
Receives a Telegram message and passes it downstream.

### 1.2 Company Name Extraction (Gemini + Structured Output)
Uses an AI agent to extract a clean `company_name` from user text into strict JSON.

### 1.3 User Acknowledgement
Immediately informs the user the process started (“I’ll look for X ads…”).

### 1.4 Ad Scraping (BrowserAct sub-workflow)
Invokes a BrowserAct workflow (“Competitor Ad Activity Monitor”) with `input-Company` set to the extracted company.

### 1.5 Analysis + Report Formatting (Gemini + Structured Output)
Analyzes scraped JSON, derives a verdict, formats an HTML report, and parses into `{ "TelegramText": "..." }`.

### 1.6 Delivery (Telegram)
Sends the HTML report back to the original Telegram chat.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Telegram)

**Overview:**  
Starts the automation when a Telegram user sends a message to the bot. Provides the raw text used for downstream extraction.

**Nodes involved:**
- **User Sends Message to Bot** (Telegram Trigger)

#### Node: User Sends Message to Bot
- **Type / role:** `telegramTrigger` — entry point; listens for incoming Telegram updates.
- **Configuration (interpreted):**
  - Update type: `message`
  - Uses Telegram bot credentials.
- **Key data produced:**
  - `message.text` (user text)
  - `message.chat.id` (reply destination)
- **Connections:**
  - Output → **Analyze user Input**
- **Edge cases / failures:**
  - Telegram credential invalid/revoked → trigger won’t receive updates.
  - Bot not configured with webhook / polling properly in n8n → no executions.
  - Messages without `text` (stickers, photos) → `$json.message.text` may be undefined; extraction prompt will receive empty/undefined input unless handled.

---

### Block 2 — Company Name Extraction (Gemini agent + structured parser)

**Overview:**  
Converts natural language (“check ads for nike”) into strict JSON `{ "company_name": "Nike" }`. This is the key input for BrowserAct scraping.

**Nodes involved:**
- **Analyze user Input** (LangChain Agent)
- **Validate inputs** (Gemini Chat Model)
- **Structured Output1** (Structured Output Parser)

#### Node: Validate inputs
- **Type / role:** `lmChatGoogleGemini` — provides the Gemini model instance to the upstream AI agent/parser chain.
- **Configuration (interpreted):**
  - Uses Google Gemini (PaLM) API credentials.
  - Default model/options (none explicitly set).
- **Connections:**
  - AI language model connection → **Analyze user Input**
  - AI language model connection → **Structured Output1**
- **Version notes:** Type version `1` (Gemini chat model node).
- **Edge cases / failures:**
  - PaLM/Gemini API credential errors, quota exhaustion, or model availability issues.
  - Latency/timeouts on large workloads (usually small here).

#### Node: Structured Output1
- **Type / role:** `outputParserStructured` — forces the agent output into a schema.
- **Configuration (interpreted):**
  - Schema example: `{ "company_name": "company_name" }`
  - `autoFix: true` (attempts to repair slightly invalid JSON outputs).
- **Connections:**
  - Output parser feeds into → **Analyze user Input** (as the agent’s output parser)
- **Edge cases / failures:**
  - If the LLM output is too malformed to fix, parsing fails and downstream nodes won’t run.
  - If user input contains multiple brands, the prompt does not define tie-breaking; model may pick the most salient.

#### Node: Analyze user Input
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — prompts the model to extract `company_name` only.
- **Configuration (interpreted):**
  - Input text: `{{ $json.message.text }}`
  - System message: strict extraction engine; must output raw JSON, `company_name` or `null`.
  - Has output parser enabled (Structured Output1).
- **Connections:**
  - Main output → **Scrape Ads from Meta and Google**
  - Main output → **Process Initialization Alert**
- **Key variables / fields used downstream:**
  - `{{$json.output.company_name}}` (note: agent output is placed under `output` by n8n LangChain agent nodes)
- **Edge cases / failures:**
  - Non-text messages: missing `message.text`.
  - If `company_name` is `null`, BrowserAct will run with empty input (may default or fail depending on BrowserAct template defaults).

---

### Block 3 — User Acknowledgement (Telegram)

**Overview:**  
Sends an immediate message to the same chat to confirm the request is being processed.

**Nodes involved:**
- **Process Initialization Alert** (Telegram)

#### Node: Process Initialization Alert
- **Type / role:** `telegram` — send message.
- **Configuration (interpreted):**
  - Text: `Hello, ok i will look for {{ $json.output.company_name }} ads. i will be rigth back.`
  - Chat ID: `{{ $('User Sends Message to Bot').first().json.message.chat.id }}`
- **Connections:**
  - No outgoing connections (terminal side-message).
- **Edge cases / failures:**
  - If trigger output is missing (unexpected execution context), the node can’t resolve the chat id.
  - Typos in message are cosmetic (“rigth”).
  - Telegram rate limits are possible in high volume.

---

### Block 4 — Ad Scraping (BrowserAct sub-workflow invocation)

**Overview:**  
Invokes a BrowserAct workflow that scrapes competitor ads from Meta and Google, using the extracted `company_name` as the primary input.

**Nodes involved:**
- **Scrape Ads from Meta and Google** (BrowserAct)

#### Node: Scrape Ads from Meta and Google
- **Type / role:** `browserAct` — runs a BrowserAct “WORKFLOW”.
- **Configuration (interpreted):**
  - Mode: `WORKFLOW`
  - BrowserAct workflow ID: `70626184596326140`
  - Input mapping:
    - `input-Company` = `{{ $json.output.company_name }}`
  - Other inputs exist in schema but are marked removed (`Facebook_Ads`, `Google_ads`), so only Company is actively mapped.
  - Incognito mode: `false`
- **Connections:**
  - Main output → **Process Ad Data**
- **Sub-workflow reference:**
  - BrowserAct template/workflow expected: **Competitor Ad Activity Monitor** (per sticky note documentation)
- **Edge cases / failures:**
  - Invalid BrowserAct API key or workflow ID → execution fails.
  - Browser automation failures (site layout changes, captchas, regional blocks).
  - If `company_name` is empty/null, BrowserAct may:
    - Use its own default (if defined), or
    - Return empty results, or
    - Error depending on template validation.
  - Runtime variability: scraping can be slow; consider n8n execution timeouts.

---

### Block 5 — Analysis + Report Formatting (Gemini agent + structured parser)

**Overview:**  
Takes scraped ad JSON, counts ads, infers activity status, extracts the best hook/copy, and returns a Telegram-safe HTML report inside a structured JSON envelope `{ "TelegramText": "..." }`.

**Nodes involved:**
- **Process Ad Data** (LangChain Agent)
- **Gemini** (Gemini Chat Model)
- **Structured Output** (Structured Output Parser)

#### Node: Gemini
- **Type / role:** `lmChatGoogleGemini` — provides the Gemini model instance for the analysis agent/parser chain.
- **Configuration (interpreted):**
  - Uses the same Gemini (PaLM) credentials.
  - Default options.
- **Connections:**
  - AI language model connection → **Process Ad Data**
  - AI language model connection → **Structured Output**
- **Edge cases / failures:**
  - Quota, auth, model errors; latency/timeouts if BrowserAct returns very large JSON.

#### Node: Structured Output
- **Type / role:** `outputParserStructured` — ensures final output is valid JSON for the Telegram sender.
- **Configuration (interpreted):**
  - Schema example: `{ "TelegramText": "TelegramText" }`
  - `autoFix: true`
- **Connections:**
  - Output parser → **Process Ad Data** (as its output parser)
- **Edge cases / failures:**
  - If LLM returns HTML only (as instructed) but not wrapped into the requested JSON, the parser must “autoFix”; if it can’t, the workflow stops before sending to Telegram.
  - HTML must be within Telegram supported tags; model might occasionally output unsupported tags or exceed 3000 chars.

#### Node: Process Ad Data
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — analysis + formatting.
- **Configuration (interpreted):**
  - Input text: `{{ $json.output.string }}`
    - Assumption: BrowserAct returns an `output.string` field containing JSON serialized as a string.
  - System message:
    - Marketing analyst logic (counts ads, checks `total_active_time`, decides “Active & Scaling” etc.)
    - Verdict rules (<24h and >3 ads ⇒ ADVERTISE NOW)
    - Output must be Telegram-safe HTML and under 3000 chars
    - Then: “send it to output using this JSON: { "TelegramText": "TelegramText" }”
  - Has output parser enabled (Structured Output).
- **Connections:**
  - Main output → **Send Message to User Chat**
- **Edge cases / failures:**
  - If BrowserAct output isn’t in `output.string` (different structure), analysis input becomes empty and results degrade.
  - If ads list is not a list, counting logic may be inconsistent.
  - If `total_active_time` formats vary (e.g., “2 days”, “1 hr”, “14hrs”), the heuristic may misclassify.

---

### Block 6 — Delivery (Telegram)

**Overview:**  
Sends the final formatted HTML intelligence report back to the same Telegram chat.

**Nodes involved:**
- **Send Message to User Chat** (Telegram)

#### Node: Send Message to User Chat
- **Type / role:** `telegram` — send message.
- **Configuration (interpreted):**
  - Text: `{{ $json.output.TelegramText }}`
  - Parse mode: `HTML`
  - Chat ID intends to reference the trigger chat:
    - Expression contains a confusing prefix:  
      `=parameters.chatId=={{ $('User Sends Message to Bot').first().json.message.chat.id }} ...`
    - Practically, the correct/needed value is:  
      `{{ $('User Sends Message to Bot').first().json.message.chat.id }}`
- **Connections:**
  - No outgoing connections (final node).
- **Edge cases / failures:**
  - **High risk:** Chat ID expression appears malformed and may cause runtime evaluation errors or an invalid chat_id.
  - If HTML contains unsupported tags/entities, Telegram may reject the message.
  - If message exceeds Telegram limits, sending fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Receive Telegram messages (entry point) | — | Analyze user Input | ### 🔍 Step 1: Input & Extraction<br><br>The workflow triggers when a user sends a brand name via Telegram. An AI agent extracts the clean company name from natural language.BrowserAct  scrapes active ad counts, copy, and run durations. |
| Analyze user Input | @n8n/n8n-nodes-langchain.agent | Extract company name into structured JSON | User Sends Message to Bot | Scrape Ads from Meta and Google; Process Initialization Alert | ### 🔍 Step 1: Input & Extraction<br><br>The workflow triggers when a user sends a brand name via Telegram. An AI agent extracts the clean company name from natural language.BrowserAct  scrapes active ad counts, copy, and run durations. |
| Validate inputs | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Gemini model for extraction | — (AI connection) | Analyze user Input; Structured Output1 |  |
| Structured Output1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce `{company_name}` JSON output | Validate inputs (AI) | Analyze user Input (parser) |  |
| Process Initialization Alert | telegram | Notify user processing started | Analyze user Input | — | ### 🔍 Step 1: Input & Extraction<br><br>The workflow triggers when a user sends a brand name via Telegram. An AI agent extracts the clean company name from natural language.BrowserAct  scrapes active ad counts, copy, and run durations. |
| Scrape Ads from Meta and Google | browserAct | Run BrowserAct scraping workflow with company input | Analyze user Input | Process Ad Data | ### 🔍 Step 1: Input & Extraction<br><br>The workflow triggers when a user sends a brand name via Telegram. An AI agent extracts the clean company name from natural language.BrowserAct  scrapes active ad counts, copy, and run durations. |
| Process Ad Data | @n8n/n8n-nodes-langchain.agent | Analyze ad data + generate HTML report | Scrape Ads from Meta and Google | Send Message to User Chat | ### 📊 Step 3: Analysis & Report Delivery<br><br>AI processes the raw data. It counts active ads, identifies the best creative hooks, and issues a final strategic verdict (ADVERTISE NOW vs. WAIT) based on competitor spending trends. The final intelligence report is formatted into clean HTML and sent directly back to the user's Telegram chat for immediate review. |
| Gemini | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Gemini model for analysis/report | — (AI connection) | Process Ad Data; Structured Output |  |
| Structured Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce `{TelegramText}` JSON output | Gemini (AI) | Process Ad Data (parser) |  |
| Send Message to User Chat | telegram | Send final HTML report to Telegram chat | Process Ad Data | — | ### 📊 Step 3: Analysis & Report Delivery<br><br>AI processes the raw data. It counts active ads, identifies the best creative hooks, and issues a final strategic verdict (ADVERTISE NOW vs. WAIT) based on competitor spending trends. The final intelligence report is formatted into clean HTML and sent directly back to the user's Telegram chat for immediate review. |
| Sticky Note | stickyNote | YouTube reference | — | — | @[youtube](ZV8ERteG_04) |
| Documentation | stickyNote | Setup notes + links | — | — | ## ⚡ Workflow Overview & Setup<br><br>**Summary:** This automation allows users to monitor competitor ad activity on Meta and Google via Telegram, providing an AI-driven strategic verdict on whether to launch competing ads.<br><br>### Requirements<br>* **Credentials:** Telegram, BrowserAct, Google Gemini (PaLM).<br>* **Mandatory:** BrowserAct API (Template: **Competitor Ad Activity Monitor**)<br><br>### How to Use<br>1. **Credentials:** Set up your Telegram Bot, BrowserAct, and Google Gemini credentials.<br>2. **BrowserAct Template:** Ensure you have the **Competitor Ad Activity Monitor** template saved in your BrowserAct account.<br>3. **Interaction:** Send a message to your bot like "Check ads for [Company Name]" to start the report generation.<br><br>### Need Help?<br>[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)<br>[How to Connect n8n to BrowserAct](https://docs.browseract.com)<br>[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | stickyNote | Explains input/extraction/scrape | — | — | ### 🔍 Step 1: Input & Extraction<br><br>The workflow triggers when a user sends a brand name via Telegram. An AI agent extracts the clean company name from natural language.BrowserAct  scrapes active ad counts, copy, and run durations. |
| Step 3 Explanation | stickyNote | Explains analysis + delivery | — | — | ### 📊 Step 3: Analysis & Report Delivery<br><br>AI processes the raw data. It counts active ads, identifies the best creative hooks, and issues a final strategic verdict (ADVERTISE NOW vs. WAIT) based on competitor spending trends. The final intelligence report is formatted into clean HTML and sent directly back to the user's Telegram chat for immediate review. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Monitor competitor ad activity via Telegram using BrowserAct & Gemini**
- (Optional) Keep execution order default (`v1`).

2) **Add Telegram Trigger**
- Node: **Telegram Trigger**  
- Name: **User Sends Message to Bot**
- Updates: `message`
- Credentials: connect your **Telegram Bot** token.
- This is the entry node.

3) **Add Gemini Chat Model (for extraction)**
- Node: **Google Gemini Chat Model** (LangChain)
- Name: **Validate inputs**
- Credentials: **Google Gemini (PaLM) API**
- Leave options default.

4) **Add Structured Output Parser (company schema)**
- Node: **Structured Output Parser** (LangChain)
- Name: **Structured Output1**
- Schema example: `{ "company_name": "company_name" }`
- Enable **Auto-fix**.

5) **Add AI Agent (company extraction)**
- Node: **AI Agent** (LangChain)
- Name: **Analyze user Input**
- Text input expression: `{{ $json.message.text }}`
- Prompt type: “Define”
- System message: (use the workflow’s extraction instructions: strict JSON only; `company_name` or `null`)
- In the Agent’s **Language Model** connection, connect **Validate inputs** to the agent.
- Enable **Output Parser** and connect **Structured Output1** as the parser.

6) **Add Telegram “processing started” message**
- Node: **Telegram**
- Name: **Process Initialization Alert**
- Chat ID expression: `{{ $('User Sends Message to Bot').first().json.message.chat.id }}`
- Text: `Hello, ok i will look for {{ $json.output.company_name }} ads. i will be rigth back.`
- Connect from **Analyze user Input** → **Process Initialization Alert**.

7) **Add BrowserAct node to run the scraping workflow**
- Node: **BrowserAct**
- Name: **Scrape Ads from Meta and Google**
- Type: `WORKFLOW`
- Workflow ID: set to your BrowserAct workflow id (expected template: **Competitor Ad Activity Monitor**)
- Map input parameter:
  - `input-Company` = `{{ $json.output.company_name }}`
- Credentials: BrowserAct API key/account.
- Connect **Analyze user Input** → **Scrape Ads from Meta and Google**.

8) **Add Gemini Chat Model (for analysis)**
- Node: **Google Gemini Chat Model** (LangChain)
- Name: **Gemini**
- Credentials: same Gemini (PaLM) API.
- Leave options default.

9) **Add Structured Output Parser (TelegramText schema)**
- Node: **Structured Output Parser** (LangChain)
- Name: **Structured Output**
- Schema example: `{ "TelegramText": "TelegramText" }`
- Enable **Auto-fix**.

10) **Add AI Agent (analysis + HTML report)**
- Node: **AI Agent** (LangChain)
- Name: **Process Ad Data**
- Text input: `{{ $json.output.string }}`
- System message: marketing analysis rules + HTML template + “Output ONLY HTML” + “wrap into JSON with TelegramText” (as in the provided workflow).
- Connect **Gemini** as the agent’s **Language Model**.
- Enable output parser and connect **Structured Output**.

11) **Add Telegram final sender**
- Node: **Telegram**
- Name: **Send Message to User Chat**
- Parse mode: `HTML`
- Chat ID expression (recommended): `{{ $('User Sends Message to Bot').first().json.message.chat.id }}`
- Text expression: `{{ $json.output.TelegramText }}`
- Connect **Process Ad Data** → **Send Message to User Chat**.

12) **(Recommended fix) Validate `company_name` before scraping**
- Add an **IF** node after **Analyze user Input** to check `{{$json.output.company_name}}` is not empty.
- If empty: send Telegram message asking user to provide a company name; do not call BrowserAct.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| @[youtube](ZV8ERteG_04) | Video reference from sticky note |
| Requirements: Telegram, BrowserAct, Google Gemini (PaLM). Mandatory BrowserAct template: “Competitor Ad Activity Monitor”. | From the “Documentation” sticky note |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |