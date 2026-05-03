Get AI crypto price analysis via Telegram using GPT-4o-mini and TwelveData

https://n8nworkflows.xyz/workflows/get-ai-crypto-price-analysis-via-telegram-using-gpt-4o-mini-and-twelvedata-12552


# Get AI crypto price analysis via Telegram using GPT-4o-mini and TwelveData

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow receives a Telegram message, uses GPT‑4o‑mini to classify whether the user is asking for a crypto price check, fetches OHLC market data from TwelveData, asks GPT‑4o‑mini to generate a structured trend analysis, formats a Telegram-ready message via GPT‑4o‑mini, and sends it back to the same Telegram chat. If any node fails during execution, an error trigger sends a notification email via Gmail.

**Target use cases**
- Telegram “crypto bot” that returns AI-generated market commentary based on recent OHLC data.
- Lightweight intent classification (“is this a price request?”) before calling paid/external APIs.

**Logical blocks**
1.1 **Telegram Intake & Normalization** → trigger and extract user query + chat identifier.  
1.2 **Intent Classification (AI)** → GPT‑4o‑mini agent produces structured intent, parsed to JSON.  
1.3 **Routing** → IF node decides whether to proceed with OHLC fetch (only “price check” path is wired).  
1.4 **Market Data Acquisition** → HTTP request to TwelveData to retrieve OHLC series.  
1.5 **AI Trend Analysis (structured)** → GPT‑4o‑mini agent turns OHLC into structured analysis JSON.  
1.6 **Message Formatting & Delivery** → prepares final message and sends to Telegram.  
1.7 **Global Error Handling** → error trigger emails failure details via Gmail.

---

## 2. Block-by-Block Analysis

### 1.1 Telegram Intake & Normalization

**Overview:** Listens for Telegram messages and prepares normalized fields (query text, chat id) for downstream AI and messaging nodes.

**Nodes involved**
- Telegram Trigger
- Extract Query & Chat ID

**Node details**

**Telegram Trigger**
- **Type / role:** `n8n-nodes-base.telegramTrigger` — Entry point via Telegram webhook updates.
- **Configuration (interpreted):** Uses Telegram credentials (bot token) and a webhook. Webhook identifier shown as `telegram-crypto-bot`.
- **Key data typically available:** `message.text`, `message.chat.id`, sender info, update metadata (depends on Telegram update type).
- **Connections:** Outputs to **Extract Query & Chat ID**.
- **Potential failures / edge cases:**
  - Telegram credential/token invalid or revoked.
  - Webhook not set / blocked by network.
  - Non-text updates (stickers, images, joins) may not include `message.text`; downstream nodes must handle missing text.

**Extract Query & Chat ID**
- **Type / role:** `n8n-nodes-base.set` — Normalizes incoming payload into predictable fields.
- **Configuration (interpreted):** Parameters are empty in the provided JSON, which usually means it has not been configured yet or was cleared. Intended behavior (based on name) is to map:
  - `chatId` ← Telegram chat id
  - `query` ← message text
- **Connections:** Receives from **Telegram Trigger**, outputs to **Classify Intent with AI**.
- **Potential failures / edge cases:**
  - If not configured, downstream AI node may not receive expected fields (classification prompt may be empty).
  - Missing `message.text` for non-text updates.
- **Version notes:** Set node `typeVersion: 1` (older UI); field mapping behavior is stable across n8n versions, but UI differs.

---

### 1.2 Intent Classification (AI)

**Overview:** Uses a LangChain Agent with GPT‑4o‑mini to classify the user’s intent and parses the result into a structured JSON object.

**Nodes involved**
- Classify Intent with AI
- OpenAI GPT-4o-mini
- Parse Intent JSON

**Node details**

**OpenAI GPT-4o-mini**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Chat LLM provider for the agent.
- **Configuration (interpreted):** Model is intended to be GPT‑4o‑mini (as per node name). Credentials required: OpenAI API key.
- **Connections:** Provides `ai_languageModel` input to **Classify Intent with AI**.
- **Potential failures / edge cases:**
  - Missing/invalid OpenAI credentials.
  - Rate limits, timeouts, model unavailability.
  - Token/context issues if the prompt is too large (unlikely here unless you include large chat history).

**Parse Intent JSON**
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces a structured schema for agent output.
- **Configuration (interpreted):** Parameters are empty in JSON; normally you define a schema (e.g., `{ intent: "price_check"|"other", symbol: "BTC/USD", interval: "1h", ... }`).
- **Connections:** Feeds into **Classify Intent with AI** via `ai_outputParser`.
- **Potential failures / edge cases:**
  - If schema isn’t configured, parsing may be ineffective or fail depending on node defaults.
  - Model may output non-JSON; parser will throw an error (then error trigger fires).

**Classify Intent with AI**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Runs an agent chain to classify the message.
- **Configuration (interpreted):** Parameters empty in JSON; typically includes system/user instructions referencing the normalized `query` field.
- **Key variables (expected):** would reference something like `{{$json.query}}`.
- **Inputs:**
  - Main input: from **Extract Query & Chat ID**
  - `ai_languageModel`: from **OpenAI GPT-4o-mini**
  - `ai_outputParser`: from **Parse Intent JSON**
- **Outputs:** Main output to **Is Price Check Intent?**
- **Potential failures / edge cases:**
  - Empty prompt due to missing `query`.
  - Parser mismatch if agent response deviates.
  - Any OpenAI API errors propagate.

---

### 1.3 Routing

**Overview:** Checks whether the message is a price check intent; if yes, proceeds to market data fetch. The “no” path is not connected in this workflow, so non-matching intents will currently end silently.

**Nodes involved**
- Is Price Check Intent?

**Node details**

**Is Price Check Intent?**
- **Type / role:** `n8n-nodes-base.if` — Branching based on a boolean condition.
- **Configuration (interpreted):** Parameters empty in JSON; intended to check a field from intent classification (e.g., `intent === "price_check"`).
- **Connections:** True (first output) goes to **Fetch OHLC Data from TwelveData**. No visible false-branch wiring in connections.
- **Potential failures / edge cases:**
  - If condition is not configured, default behavior may always be false or error depending on node state.
  - If intent field path doesn’t exist, condition evaluation may fail or resolve unexpectedly.
  - Missing “else” handling: users may receive no response for other intents.

---

### 1.4 Market Data Acquisition

**Overview:** Pulls OHLC time series data from TwelveData using HTTP Request for the requested symbol/timeframe.

**Nodes involved**
- Fetch OHLC Data from TwelveData

**Node details**

**Fetch OHLC Data from TwelveData**
- **Type / role:** `n8n-nodes-base.httpRequest` — Calls TwelveData REST API.
- **Configuration (interpreted):** Parameters empty in JSON; expected configuration:
  - Method: GET
  - URL like `https://api.twelvedata.com/time_series`
  - Query parameters: `symbol`, `interval`, `apikey`, `outputsize`, possibly `format=JSON`
- **Connections:** Output to **Analyze Price Trends with AI**
- **Potential failures / edge cases:**
  - API key missing/invalid.
  - TwelveData rate limits / quota exhaustion.
  - Symbol formatting issues (e.g., `BTC/USD` vs `BTCUSDT` depending on provider expectations).
  - Response shape changes; missing OHLC fields; empty dataset on unsupported interval.
  - Network errors/timeouts.
- **Version notes:** HTTP Request node `typeVersion: 3`—supports modern auth options and response handling.

---

### 1.5 AI Trend Analysis (structured)

**Overview:** Converts the raw OHLC series into a structured analysis JSON (trend direction, key levels, volatility, summary, etc.), using GPT‑4o‑mini plus a structured output parser.

**Nodes involved**
- Analyze Price Trends with AI
- OpenAI GPT-4o-mini Analysis
- Parse Analysis JSON
- Structure Analysis Data

**Node details**

**OpenAI GPT-4o-mini Analysis**
- **Type / role:** `lmChatOpenAi` — LLM provider dedicated to analysis stage.
- **Configuration (interpreted):** GPT‑4o‑mini; OpenAI credentials required.
- **Connections:** Supplies `ai_languageModel` to **Analyze Price Trends with AI**.
- **Potential failures:** Same as other OpenAI node (auth/rate limits/timeouts).

**Parse Analysis JSON**
- **Type / role:** `outputParserStructured` — Validates and parses the analysis output into a predictable structure.
- **Configuration (interpreted):** Empty in JSON; intended to define schema for analysis output (e.g., `trend`, `support`, `resistance`, `signals`, `risk_notes`).
- **Connections:** Feeds `ai_outputParser` into **Analyze Price Trends with AI**.
- **Potential failures:** Model output not matching schema → parse error.

**Analyze Price Trends with AI**
- **Type / role:** `agent` — Generates analysis from OHLC data.
- **Configuration (interpreted):** Empty in JSON; typically includes instructions referencing OHLC array from TwelveData response (e.g., `{{$json.values}}`).
- **Inputs:**
  - Main: from **Fetch OHLC Data from TwelveData**
  - `ai_languageModel`: from **OpenAI GPT-4o-mini Analysis**
  - `ai_outputParser`: from **Parse Analysis JSON**
- **Outputs:** Main output to **Structure Analysis Data**
- **Edge cases:**
  - OHLC payload too large → token pressure; mitigate by limiting outputsize or pre-aggregating.
  - Incomplete OHLC series leading to low-confidence analysis; should instruct the model to state limitations.

**Structure Analysis Data**
- **Type / role:** `n8n-nodes-base.set` — Maps parsed analysis fields to a clean structure used by formatter/sender.
- **Configuration (interpreted):** Parameters empty in JSON; intended to create fields like:
  - `chatId` (carry from earlier)
  - `analysis` object fields
  - `symbol`, `interval`, `timestamp`
- **Connections:** Outputs to **Format Message for Telegram**
- **Potential failures / edge cases:**
  - Missing `chatId` if not preserved across steps (common issue). You may need to merge original Telegram data back in or pass it through all nodes.
  - Field path mistakes from parsed JSON.

---

### 1.6 Message Formatting & Delivery

**Overview:** Uses GPT‑4o‑mini to turn structured analysis into a short, readable Telegram message, then sends it back to the originating chat.

**Nodes involved**
- Format Message for Telegram
- OpenAI GPT-4o-mini Formatter
- Send Analysis to Telegram

**Node details**

**OpenAI GPT-4o-mini Formatter**
- **Type / role:** `lmChatOpenAi` — LLM used specifically for message formatting.
- **Configuration (interpreted):** GPT‑4o‑mini; OpenAI credentials.
- **Connections:** Supplies `ai_languageModel` to **Format Message for Telegram**.

**Format Message for Telegram**
- **Type / role:** `agent` — Produces final text for Telegram from structured fields.
- **Configuration (interpreted):** Empty in JSON; typically includes constraints like max length, Markdown formatting, bullet points, and safety notes (“not financial advice”).
- **Inputs:**
  - Main: from **Structure Analysis Data**
  - `ai_languageModel`: from **OpenAI GPT-4o-mini Formatter**
- **Outputs:** Main output to **Send Analysis to Telegram**
- **Potential failures / edge cases:**
  - Telegram Markdown parse issues if formatting is invalid; consider using MarkdownV2 escaping or plain text.
  - If formatter doesn’t output the exact field the Telegram node expects, message may be empty.

**Send Analysis to Telegram**
- **Type / role:** `n8n-nodes-base.telegram` — Sends a message to Telegram chat.
- **Configuration (interpreted):** Parameters empty in JSON; expected:
  - Operation: Send Message
  - Chat ID: from `chatId`
  - Text: formatted message from formatter output
  - Optional: parse mode (Markdown/HTML)
- **Connections:** Terminal node in main flow.
- **Potential failures / edge cases:**
  - Telegram credential/token issues.
  - Chat id missing/incorrect.
  - Message too long (Telegram limits).
  - Parse mode errors due to formatting.

---

### 1.7 Global Error Handling

**Overview:** Catches workflow execution errors and emails details via Gmail.

**Nodes involved**
- Workflow Error Handler
- Send Error Email

**Node details**

**Workflow Error Handler**
- **Type / role:** `n8n-nodes-base.errorTrigger` — Runs when the workflow errors.
- **Configuration:** Default; triggers on errors in this workflow.
- **Connections:** Outputs to **Send Error Email**.
- **Edge cases:**
  - Only triggers for execution failures; logical failures (e.g., IF false branch) won’t trigger.
  - If the error happens before key context fields exist, email content must handle missing references.

**Send Error Email**
- **Type / role:** `n8n-nodes-base.gmail` — Sends an email notification.
- **Configuration (interpreted):** Parameters empty in JSON; expected:
  - Resource: Message
  - Operation: Send
  - To/Subject/Body: include error details from Error Trigger payload
- **Credentials:** Gmail OAuth2 credentials.
- **Potential failures / edge cases:**
  - OAuth token expired/needs re-consent.
  - Gmail API scopes insufficient.
  - If error payload is large, email body may need truncation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Comment/annotation | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment/annotation | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment/annotation | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment/annotation | — | — |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Comment/annotation | — | — |  |
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Receives Telegram messages (entry) | — | Extract Query & Chat ID |  |
| Extract Query & Chat ID | n8n-nodes-base.set | Normalize query + chat id | Telegram Trigger | Classify Intent with AI |  |
| Classify Intent with AI | @n8n/n8n-nodes-langchain.agent | AI intent classification | Extract Query & Chat ID (+ OpenAI + Parser) | Is Price Check Intent? |  |
| OpenAI GPT-4o-mini | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for intent classifier | — | Classify Intent with AI (ai_languageModel) |  |
| Parse Intent JSON | @n8n/n8n-nodes-langchain.outputParserStructured | Structured output parser for intent | — | Classify Intent with AI (ai_outputParser) |  |
| Is Price Check Intent? | n8n-nodes-base.if | Branch: proceed only if price intent | Classify Intent with AI | Fetch OHLC Data from TwelveData |  |
| Fetch OHLC Data from TwelveData | n8n-nodes-base.httpRequest | Pull OHLC series from TwelveData | Is Price Check Intent? | Analyze Price Trends with AI |  |
| Analyze Price Trends with AI | @n8n/n8n-nodes-langchain.agent | AI analysis from OHLC | Fetch OHLC Data (+ OpenAI + Parser) | Structure Analysis Data |  |
| OpenAI GPT-4o-mini Analysis | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for OHLC analysis | — | Analyze Price Trends with AI (ai_languageModel) |  |
| Parse Analysis JSON | @n8n/n8n-nodes-langchain.outputParserStructured | Structured output parser for analysis | — | Analyze Price Trends with AI (ai_outputParser) |  |
| Structure Analysis Data | n8n-nodes-base.set | Prepare fields for formatter + sender | Analyze Price Trends with AI | Format Message for Telegram |  |
| Format Message for Telegram | @n8n/n8n-nodes-langchain.agent | Create Telegram message text | Structure Analysis Data (+ OpenAI) | Send Analysis to Telegram |  |
| OpenAI GPT-4o-mini Formatter | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for final formatting | — | Format Message for Telegram (ai_languageModel) |  |
| Send Analysis to Telegram | n8n-nodes-base.telegram | Send Telegram message | Format Message for Telegram | — |  |
| Workflow Error Handler | n8n-nodes-base.errorTrigger | Catch workflow errors | — | Send Error Email |  |
| Send Error Email | n8n-nodes-base.gmail | Email error notification | Workflow Error Handler | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: **Get crypto price analysis via Telegram with AI and TwelveData**
- Keep execution order: default (`v1`).

2) **Add Telegram Trigger**
- Node: **Telegram Trigger**
- Credentials: Telegram bot token (create bot via BotFather).
- Trigger updates: “On message” (default).
- Activate webhook (n8n will provide URL; Telegram must reach it).

3) **Add “Set” node: Extract Query & Chat ID**
- Node: **Set**
- Create fields (recommended):
  - `chatId` = `{{$json.message.chat.id}}`
  - `query` = `{{$json.message.text}}`
  - (Optional) `username` = `{{$json.message.from.username}}`
- Connect: **Telegram Trigger → Extract Query & Chat ID**

4) **Add OpenAI chat model (for intent)**
- Node: **OpenAI Chat Model** (`lmChatOpenAi`)
- Model: **gpt-4o-mini**
- Credentials: OpenAI API key.

5) **Add Structured Output Parser (for intent)**
- Node: **Structured Output Parser**
- Define a schema (example):
  - `intent` (string; values like `price_check` / `other`)
  - `symbol` (string; example `BTC/USD`)
  - `interval` (string; example `1h`, `1day`)
- Ensure the parser expects valid JSON.

6) **Add AI Agent: Classify Intent with AI**
- Node: **AI Agent**
- Connect:
  - Main input from **Extract Query & Chat ID**
  - `ai_languageModel` from **OpenAI GPT-4o-mini**
  - `ai_outputParser` from **Parse Intent JSON**
- Prompt guidelines (recommended):
  - “Classify the user request. If they ask for crypto price/analysis, set intent=price_check and extract symbol+interval; otherwise intent=other.”

7) **Add IF node: Is Price Check Intent?**
- Condition example:
  - `{{$json.intent}}` equals `price_check`
- Connect: **Classify Intent with AI → Is Price Check Intent?**
- Wire the **true** output forward.
- (Recommended) Add a false-branch response node to tell the user what commands are supported.

8) **Add HTTP Request: Fetch OHLC Data from TwelveData**
- Node: **HTTP Request**
- Method: GET
- URL: `https://api.twelvedata.com/time_series`
- Query parameters (example):
  - `symbol` = `{{$json.symbol}}`
  - `interval` = `{{$json.interval}}`
  - `outputsize` = `100`
  - `apikey` = your TwelveData API key (store in n8n credentials or environment variable)
- Response: JSON.
- Connect: **Is Price Check Intent? (true) → Fetch OHLC Data from TwelveData**

9) **Add OpenAI chat model (for analysis)**
- Node: `lmChatOpenAi`
- Model: **gpt-4o-mini**
- Credentials: OpenAI.

10) **Add Structured Output Parser (for analysis)**
- Node: Structured Output Parser
- Schema example:
  - `summary` (string)
  - `trend` (string)
  - `support_levels` (array of numbers/strings)
  - `resistance_levels` (array)
  - `volatility` (string)
  - `risk_note` (string)

11) **Add AI Agent: Analyze Price Trends with AI**
- Node: AI Agent
- Inputs:
  - Main from **Fetch OHLC Data from TwelveData**
  - `ai_languageModel` from **OpenAI GPT-4o-mini Analysis**
  - `ai_outputParser` from **Parse Analysis JSON**
- Prompt recommendation:
  - Provide the OHLC array (e.g., TwelveData `values`) and ask for concise technical-style analysis with limitations.

12) **Add Set node: Structure Analysis Data**
- Node: Set
- Map/carry forward:
  - `chatId` (ensure it’s still available; if not, pass it through earlier nodes or use a Merge node)
  - `symbol`, `interval`
  - `analysis` fields from parsed analysis output

13) **Add OpenAI chat model (formatter)**
- Node: `lmChatOpenAi`
- Model: **gpt-4o-mini**
- Credentials: OpenAI.

14) **Add AI Agent: Format Message for Telegram**
- Node: AI Agent
- Main input: **Structure Analysis Data**
- `ai_languageModel`: **OpenAI GPT-4o-mini Formatter**
- Prompt constraints:
  - Produce a short Telegram message.
  - Optionally specify Markdown vs plain text.
  - Include `symbol`, `interval`, key levels, and a short disclaimer.

15) **Add Telegram node: Send Analysis to Telegram**
- Node: **Telegram**
- Operation: Send Message
- Chat ID: `{{$json.chatId}}` (or wherever you placed it)
- Text: output field from the formatter (commonly `{{$json.text}}` or similar—align with your agent output)
- Connect: **Format Message for Telegram → Send Analysis to Telegram**

16) **Add error handling**
- Node: **Error Trigger** (Workflow Error Handler)
- Node: **Gmail** (Send Error Email)
  - Credentials: Gmail OAuth2
  - To: your address
  - Subject/body: include error info from error trigger payload (execution, node name, message).
- Connect: **Workflow Error Handler → Send Error Email**

17) **Test**
- Send a Telegram message like: “Analyze BTC/USD on 1h”.
- Verify:
  - Intent JSON parsing works.
  - TwelveData returns values.
  - Telegram message sends successfully.
  - Force an error (e.g., wrong API key) to confirm Gmail error email.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes are present but empty in the provided workflow JSON. | No additional embedded notes/links were provided. |
| The workflow currently has many nodes with empty parameters (Set/If/HTTP/Telegram/Gmail and all Agent/Parser prompts). | You must configure these nodes for the workflow to function end-to-end. |
| The IF node’s “false” path is not connected, so non-price intents may produce no reply. | Consider adding a Telegram reply for unsupported intents. |
| Ensure `chatId` is preserved across the AI/HTTP steps. | Common fix: store `chatId` early and keep it in all Set nodes, or use a Merge node to re-join context. |