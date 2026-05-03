Analyze stocks in Warren Buffett style from Telegram with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/analyze-stocks-in-warren-buffett-style-from-telegram-with-openai-and-gmail-14843


# Analyze stocks in Warren Buffett style from Telegram with OpenAI and Gmail

### 1. Workflow Overview

**Warren Buffett Style Stock Analyzer** is an event-driven workflow that lets a user send a stock ticker symbol via a Telegram bot, triggers a multi-step AI research pipeline, and delivers a fully formatted HTML investment analysis email styled as a Warren Buffett value-investing report — complete with a BUY / HOLD / AVOID verdict.

The workflow is organized into four logical blocks:

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Input Reception & Validation | Receives the Telegram message, extracts and sanitizes the ticker, validates it against a regex pattern, and rejects invalid inputs early. |
| 2 | User Acknowledgement | Sends an immediate Markdown message back to Telegram informing the user that the analysis is in progress and will take 30–60 seconds. |
| 3 | AI Research & Analysis Engine | An AI Agent (driven by OpenAI GPT-4o-mini) uses two tools — SerpAPI for live web research and a Calculator for DCF math — to produce a complete HTML analysis document following a Buffett-style framework with seven sections and a verdict. |
| 4 | Output Validation & Delivery | Validates that the agent returned well-formed HTML (contains `</html>`), emails the report via Gmail with the verdict in the subject line, and sends a short Telegram confirmation. If the output is invalid or the agent failed, an error message is sent to Telegram instead. |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Input Reception & Validation

**Overview:** This block captures the incoming Telegram message, extracts the ticker symbol and user metadata, sanitizes the ticker, and validates it. Invalid tickers are rejected with a helpful reply before any expensive AI processing begins.

**Nodes Involved:**
- 📱 Telegram Trigger
- 🔤 Extract Ticker & User
- 🔎 Validate Ticker
- ⚠️ Invalid Ticker Reply

---

**📱 Telegram Trigger**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.telegramTrigger` (v1.1) |
| **Role** | Webhook entry point; listens for incoming messages on the Telegram bot |
| **Configuration** | Listens for `message` updates only. No additional fields configured. |
| **Credentials** | Telegram Bot API token (set in n8n credential store) |
| **Input** | None — this is the workflow trigger |
| **Output** | → 🔤 Extract Ticker & User (main) |
| **Output payload** | `json.message.text`, `json.message.chat.id`, `json.message.from.first_name`, `json.message.from.username` |
| **Edge cases** | Bot not added to chat; webhook URL not reachable; message is a sticker/image with no text (text will be `undefined` causing downstream expression errors) |

---

**🔤 Extract Ticker & User**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.set` (v3.3) |
| **Role** | Transforms raw Telegram message into four clean fields used by all downstream nodes |
| **Configuration** | Four assignments: |
| | `ticker` — `={{ $json.message.text.toUpperCase().replace(/[^A-Z0-9.]/g, '').trim() }}` — uppercases message text and strips all characters except A-Z, 0-9, and period |
| | `chat_id` — `={{ $json.message.chat.id }}` — the Telegram chat identifier for replies |
| | `username` — `={{ $json.message.from.first_name \|\| $json.message.from.username \|\| 'Investor' }}` — fallback chain for user name |
| | `recipientEmail` — `user@example.com` — **hardcoded default that must be changed** to the recipient's Gmail address |
| **Input** | ← 📱 Telegram Trigger |
| **Output** | → 🔎 Validate Ticker |
| **Key expressions** | The `ticker` expression strips all non-alphanumeric characters. A message like "buy aapl!" becomes "AAPL". A message like "hello" becomes "" (empty). |
| **Edge cases** | If `message.text` is undefined (e.g. photo message), the expression throws. The `recipientEmail` is a placeholder — leaving it as-is will send reports to a non-existent address. |

---

**🔎 Validate Ticker**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.if` (v2) |
| **Role** | Regex gate — ensures the ticker looks like a valid stock symbol before running the AI agent |
| **Configuration** | Single condition: `ticker` matches regex `^[A-Z0-9.\-]{1,10}$` (1–10 characters, uppercase letters, digits, period, hyphen). Combinator: AND (single condition). Case-sensitive, strict type validation. |
| **Input** | ← 🔤 Extract Ticker & User |
| **Output (true)** | → 📨 Send Acknowledgement |
| **Output (false)** | → ⚠️ Invalid Ticker Reply |
| **Edge cases** | Empty string (from non-text messages) fails regex → routes to error. Very long strings (>10 chars) fail → routes to error. Non-ASCII characters already stripped by the Set node, so the regex mostly catches length and format. |

---

**⚠️ Invalid Ticker Reply**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.telegram` (v1.2) |
| **Role** | Sends a user-friendly error message back to the Telegram chat when the ticker fails validation |
| **Configuration** | Static Markdown text with examples (AAPL, MSFT, GOOGL, AMZN, BRK.B). Chat ID from `$('🔤 Extract Ticker & User').item.json.chat_id`. Parse mode: Markdown. |
| **Input** | ← 🔎 Validate Ticker (false branch) |
| **Output** | None — this is a terminal node for the error path |
| **Credentials** | Telegram Bot API |
| **Edge cases** | Chat ID reference could fail if the Extract Ticker & User node produced no items. Markdown parse errors if the message contains unescaped special characters. |

---

#### Block 2 — User Acknowledgement

**Overview:** After validation passes, an immediate acknowledgement is sent to Telegram telling the user that analysis has started, what steps will be performed, and that results will arrive via email in roughly 30–60 seconds.

**Nodes Involved:**
- 📨 Send Acknowledgement

---

**📨 Send Acknowledgement**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.telegram` (v1.2) |
| **Role** | Proactive status message — sets user expectations before the AI agent begins its potentially long research |
| **Configuration** | Dynamic Markdown text referencing `$('🔤 Extract Ticker & User').item.json.ticker`. Lists analysis steps (financial statements, moat assessment, management review, DCF model, margin of safety). Chat ID from the same reference. Parse mode: Markdown. |
| **Input** | ← 🔎 Validate Ticker (true branch) |
| **Output** | → 🧠 Buffett AI Agent |
| **Credentials** | Telegram Bot API |
| **Edge cases** | If the ticker reference is empty or missing, the Markdown template will show an empty ticker name. Telegram Markdown v1 does not support all formatting; `_` for italic may fail if the text contains underscores within words. |

---

#### Block 3 — AI Research & Analysis Engine

**Overview:** This is the core of the workflow. An AI Agent node orchestrates two tools (SerpAPI web search and a Calculator) powered by OpenAI GPT-4o-mini as the language model. The agent receives a detailed system prompt instructing it to act as Warren Buffett, perform extensive web research, run DCF calculations, and produce a complete HTML document with seven analysis sections and a verdict.

**Nodes Involved:**
- 🧠 Buffett AI Agent
- 🔍 SerpAPI Web Search (tool)
- 🧮 Calculator (DCF Math) (tool)
- 🤖 OpenAI GPT-4o-mini (language model)

---

**🧠 Buffett AI Agent**

| Property | Detail |
|----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.agent` (v3.1) |
| **Role** | Central orchestrator — receives the ticker and username, uses the system prompt to guide research, tool usage, and final HTML output generation |
| **Configuration** | **Prompt text:** `=Analyze the company with ticker symbol: {{ $('🔤 Extract Ticker & User').item.json.ticker }}\n\nRequested by: {{ $('🔤 Extract Ticker & User').item.json.username }}\n\nDate of analysis: {{ $now.format('MMMM D, YYYY') }}` |
| | **Max iterations:** 15 (allows up to 15 tool-call cycles) |
| | **System message:** Extremely detailed ~3000-word prompt defining the persona (Warren Buffett), mandatory tool usage order (SerpAPI first, then Calculator, then final output), required 10 search topics, output format (complete HTML only, no markdown, inline CSS), seven-section structure with specific placeholder names, badge color rules, verdict color rules, and styling constraints |
| | **continueOnFail:** `true` — if the agent encounters an error, the workflow continues (routing to the error handler downstream) |
| | **Prompt type:** `define` (user-defined) |
| **Input connections** | Main: ← 📨 Send Acknowledgement |
| | ai_tool: ← 🔍 SerpAPI Web Search, ← 🧮 Calculator (DCF Math) |
| | ai_languageModel: ← 🤖 OpenAI GPT-4o-mini |
| **Output** | → 🔀 Check Agent Output. The agent's `json.output` field contains the raw HTML string. |
| **Key expressions** | References `$('🔤 Extract Ticker & User')` for ticker and username |
| **Edge cases** | SerpAPI rate limit or quota exhaustion may cause tool failures. OpenAI API timeout or content policy block. If the model exceeds the 15-iteration limit, it may produce incomplete output. `continueOnFail` means errors don't halt execution but the output may be malformed or contain error text. Very long HTML output may exceed node memory or API response limits. The system prompt insists on HTML-only output but the model may sometimes include markdown or preamble text. |

**System Prompt — Key Directives:**

1. **Persona:** Warren Buffett — folksy, plain-spoken, wise, self-deprecating, grounded in first principles.
2. **Mandatory Tool Usage Order:**
   - Step 1: SerpAPI — multiple focused searches for 10 specific data categories (revenue, ROE, FCF, debt, price/market cap, moat, management, risks, analyst targets, recent news).
   - Step 2: Calculator — all DCF arithmetic (growth projections, discounting, terminal value, present value, intrinsic value per share, margin of safety %).
   - Step 3: Final output — only a complete HTML document, no markdown, no explanations.
3. **Output Structure:** Seven sections (Business Understanding, Durable Competitive Moat, Management Quality, Financial Health 10-Year Review, Intrinsic Value DCF Model, Margin of Safety, Circle of Competence & Key Risks) plus a Verdict section and a disclaimer footer.
4. **Styling Rules:** Inline CSS only, Georgia/Times New Roman serif font, specific color palette (dark green #1a3a1a, cream #f5f0e8), badge colors for Strong/Moderate/Weak ratings, verdict badge colors (BUY=#2d6a2d, HOLD=#7a5c1e, AVOID=#8b2020), ROE color thresholds, FCF color logic, price vs intrinsic value color logic.

---

**🔍 SerpAPI Web Search**

| Property | Detail |
|----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.toolSerpApi` (v1) |
| **Role** | Tool available to the AI Agent for live web searches via SerpAPI (Google search results API) |
| **Configuration** | No additional options configured (uses default search behavior) |
| **Connection** | ai_tool → 🧠 Buffett AI Agent |
| **Credentials** | SerpAPI API key (set in n8n credential store) |
| **Edge cases** | Free tier limited to 100 searches/month. Rate limiting. API key missing or invalid causes tool call failures. Search results may be stale or irrelevant for obscure tickers. |

---

**🧮 Calculator (DCF Math)**

| Property | Detail |
|----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.toolCalculator` (v1) |
| **Role** | Tool available to the AI Agent for precise arithmetic — used for DCF calculations, growth projections, discounting, and margin-of-safety percentages |
| **Configuration** | No parameters (default calculator behavior) |
| **Connection** | ai_tool → 🧠 Buffett AI Agent |
| **Credentials** | None (local computation) |
| **Edge cases** | Division by zero if shares outstanding data is missing. Very large numbers may hit floating-point precision limits. The agent must formulate correct mathematical expressions; malformed expressions will cause tool errors. |

---

**🤖 OpenAI GPT-4o-mini**

| Property | Detail |
|----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3) |
| **Role** | Language model driving the AI Agent's reasoning and text generation |
| **Configuration** | Model: `gpt-4o-mini` (selected from list). No additional options or built-in tools enabled. |
| **Connection** | ai_languageModel → 🧠 Buffett AI Agent |
| **Credentials** | OpenAI API key (set in n8n credential store) |
| **Edge cases** | API key quota exhaustion. Rate limiting on high-volume usage. Content moderation may block certain outputs. gpt-4o-mini may struggle with the very long system prompt and complex HTML generation — may truncate or simplify. Token limit may be reached on the input (system prompt + tool results). |

---

#### Block 4 — Output Validation & Delivery

**Overview:** After the AI Agent completes, this block checks whether the output contains valid HTML (specifically the `</html>` closing tag). Valid output is emailed as an HTML Gmail message and a Telegram confirmation is sent. Invalid or failed output triggers a Telegram error message.

**Nodes Involved:**
- 🔀 Check Agent Output
- 📧 Send Gmail Report
- ✅ Telegram Confirmation
- ❌ Telegram Error Handler

---

**🔀 Check Agent Output**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.if` (v2) |
| **Role** | Gate that validates the agent's output is a complete HTML document before sending it via email |
| **Configuration** | Single condition: `output` (string) contains `</html>`. Combinator: AND. Case-sensitive, strict type validation. |
| **Input** | ← 🧠 Buffett AI Agent |
| **Output (true)** | → 📧 Send Gmail Report |
| **Output (false)** | → ❌ Telegram Error Handler |
| **Edge cases** | If `output` is undefined/null (agent crashed), the `.contains` check will fail → routes to error. If the agent returns valid HTML without the exact `</html>` tag (e.g. different casing or whitespace), it will incorrectly route to error. If the agent returns HTML with extra text before `<!DOCTYPE>`, it still passes the check. |

---

**📧 Send Gmail Report**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.gmail` (v2.1) |
| **Role** | Sends the full HTML analysis as an email body to the recipient specified in the Extract Ticker & User node |
| **Configuration** | |
| | **To:** `={{ $('🔤 Extract Ticker & User').item.json.recipientEmail }}` |
| | **Subject:** `=📊 Buffett Analysis: {{ $('🔤 Extract Ticker & User').item.json.ticker }} — {{ $('🧠 Buffett AI Agent').item.json.output.match(/VERDICT: (BUY\|HOLD\|AVOID)/)?.[1] \|\| 'Analysis Complete' }} \| {{ $now.format('dd/LL/yyyy') }}` — extracts the verdict from the HTML via regex and falls back to "Analysis Complete" |
| | **Message body:** `={{ $('🧠 Buffett AI Agent').item.json.output }}` — the raw HTML from the agent |
| | **Append attribution:** Disabled |
| **Input** | ← 🔀 Check Agent Output (true branch) |
| **Output** | → ✅ Telegram Confirmation |
| **Credentials** | Gmail OAuth2 (connected in n8n credential store) |
| **Edge cases** | Gmail may clip very large HTML emails (over ~102KB). Inline CSS is used but some email clients may still render differently. The regex in the subject may fail to match if the HTML uses different casing for "VERDICT". OAuth2 token expiration. Sending rate limits. Recipient address is the hardcoded placeholder unless changed. |

---

**✅ Telegram Confirmation**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.telegram` (v1.2) |
| **Role** | Informs the user via Telegram that the analysis is complete, the email has been sent, and shows the extracted verdict |
| **Configuration** | Dynamic Markdown text referencing ticker from Extract node, verdict from agent output regex (`match(/VERDICT: (BUY|HOLD|AVOID)/)?.[1] || 'See email for details'`), and chat ID. Parse mode: Markdown. |
| **Input** | ← 📧 Send Gmail Report |
| **Output** | None — terminal node for the success path |
| **Credentials** | Telegram Bot API |
| **Edge cases** | Regex extraction may fail if the HTML format deviates from the template. Markdown formatting may break if the verdict text contains special characters. |

---

**❌ Telegram Error Handler**

| Property | Detail |
|----------|--------|
| **Type** | `n8n-nodes-base.telegram` (v1.2) |
| **Role** | Sends an error message to Telegram when the AI agent output is invalid or the agent failed |
| **Configuration** | Dynamic Markdown text including the ticker, a troubleshooting checklist, and a truncated (first 200 chars) of the agent's output or error. Chat ID from Extract node. Parse mode: Markdown. |
| **Input** | ← 🔀 Check Agent Output (false branch) |
| **Output** | None — terminal node for the error path |
| **Credentials** | Telegram Bot API |
| **Edge cases** | The `output?.substring(0, 200)` may expose raw HTML or error stack traces to the user. If `output` is completely undefined, the fallback "No details available" is shown. Chat ID reference may fail if the upstream data was lost. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📱 Telegram Trigger | telegramTrigger (v1.1) | Webhook entry point — listens for Telegram messages | — | 🔤 Extract Ticker & User | 📌 Overview: 🏦 Warren Buffett Style Stock Analyzer — Send any stock ticker to your Telegram bot (e.g. AAPL, MSFT, AMZN) and receive a full value investing analysis in your Gmail inbox — written in Warren Buffett's voice. The AI agent researches 10 data points in real-time (financials, moat, management, DCF model, margin of safety) then delivers a formatted HTML report with a BUY / HOLD / AVOID verdict. ; 📌 Setup Required: ⚙️ Required Setup — 1. Credentials to connect: 📱 Telegram Bot API, 🔍 SerpAPI (web search), 🤖 OpenAI API, 📧 Gmail OAuth2 ; 📌 Step 1 - Input: 📥 Step 1 - Input — 1. User sends ticker via Telegram 2. Ticker is sanitized (uppercase, non-alpha stripped) 3. Config node supplies recipient email 4. Validation checks ticker is 1–10 chars 5. Acknowledgement sent (sets expectations) 💡 Tip: The bot also works for international tickers (e.g. ASML, TSM) |
| 🔤 Extract Ticker & User | set (v3.3) | Extracts and sanitizes ticker, chat ID, username, and recipient email from the Telegram message | 📱 Telegram Trigger | 🔎 Validate Ticker | 📌 Step 1 - Input: 📥 Step 1 - Input — 1. User sends ticker via Telegram 2. Ticker is sanitized (uppercase, non-alpha stripped) 3. Config node supplies recipient email 4. Validation checks ticker is 1–10 chars 5. Acknowledgement sent (sets expectations) 💡 Tip: The bot also works for international tickers (e.g. ASML, TSM) ; 📌 Setup Required: ⚙️ Required Setup — 2. Set your email → Open 🔤 Extract Ticker & User → Change recipientEmail value to your Gmail ; 📌 Email Config: ✉️ Email Recipient — Set your email in the 🔤 Extract Ticker & User node — look for the recipientEmail field. Change REPLACE_WITH_YOUR_GMAIL@gmail.com to your actual Gmail address. The Gmail account sending the report must be connected via OAuth2 in credentials. |
| 🔎 Validate Ticker | if (v2) | Regex validation gate — ensures ticker is 1–10 chars of valid stock symbol characters | 🔤 Extract Ticker & User | 📨 Send Acknowledgement (true), ⚠️ Invalid Ticker Reply (false) | 📌 Step 1 - Input: 📥 Step 1 - Input — 1. User sends ticker via Telegram 2. Ticker is sanitized (uppercase, non-alpha stripped) 3. Config node supplies recipient email 4. Validation checks ticker is 1–10 chars 5. Acknowledgement sent (sets expectations) 💡 Tip: The bot also works for international tickers (e.g. ASML, TSM) ; 📌 Error Handling: ❌ Error Handling — Two error paths are covered: Path A — Invalid ticker (too short/long) → Caught before agent runs → Sends friendly Telegram hint ; Path B — Agent failure or bad output → continueOnFail keeps workflow alive → IF node checks for valid HTML → Routes to error message node |
| ⚠️ Invalid Ticker Reply | telegram (v1.2) | Sends a helpful error message when the ticker fails validation | 🔎 Validate Ticker (false) | — | 📌 Error Handling: ❌ Error Handling — Two error paths are covered: Path A — Invalid ticker (too short/long) → Caught before agent runs → Sends friendly Telegram hint ; Path B — Agent failure or bad output → continueOnFail keeps workflow alive → IF node checks for valid HTML → Routes to error message node |
| 📨 Send Acknowledgement | telegram (v1.2) | Sends an immediate status message telling the user analysis is underway | 🔎 Validate Ticker (true) | 🧠 Buffett AI Agent | 📌 Step 1 - Input: 📥 Step 1 - Input — 1. User sends ticker via Telegram 2. Ticker is sanitized (uppercase, non-alpha stripped) 3. Config node supplies recipient email 4. Validation checks ticker is 1–10 chars 5. Acknowledgement sent (sets expectations) 💡 Tip: The bot also works for international tickers (e.g. ASML, TSM) |
| 🧠 Buffett AI Agent | @n8n/n8n-nodes-langchain.agent (v3.1) | Central AI orchestrator — uses tools to research and generate a complete Buffett-style HTML analysis | 📨 Send Acknowledgement (main), 🔍 SerpAPI Web Search (ai_tool), 🧮 Calculator (ai_tool), 🤖 OpenAI GPT-4o-mini (ai_languageModel) | 🔀 Check Agent Output | 📌 Step 2 - AI Engine: 🧠 Step 2 - AI Engine — Agent tools (for qualitative research): 🔍 SerpAPI — moat, management, news, analyst opinions ; 🧮 Calculator — precise DCF arithmetic |
| 🔍 SerpAPI Web Search | @n8n/n8n-nodes-langchain.toolSerpApi (v1) | Tool for live web search available to the AI Agent | — | 🧠 Buffett AI Agent (ai_tool) | 📌 Step 2 - AI Engine: 🧠 Step 2 - AI Engine — Agent tools (for qualitative research): 🔍 SerpAPI — moat, management, news, analyst opinions ; 🧮 Calculator — precise DCF arithmetic ; 📌 Setup Required: ⚙️ Required Setup — 1. Credentials to connect: 🔍 SerpAPI (web search) ; 4. SerpAPI free tier: 100 searches/month free — Sign up at serpapi.com |
| 🧮 Calculator (DCF Math) | @n8n/n8n-nodes-langchain.toolCalculator (v1) | Tool for precise arithmetic available to the AI Agent (DCF calculations) | — | 🧠 Buffett AI Agent (ai_tool) | 📌 Step 2 - AI Engine: 🧠 Step 2 - AI Engine — Agent tools (for qualitative research): 🔍 SerpAPI — moat, management, news, analyst opinions ; 🧮 Calculator — precise DCF arithmetic |
| 🤖 OpenAI GPT-4o-mini | @n8n/n8n-nodes-langchain.lmChatOpenAi (v1.3) | Language model powering the AI Agent's reasoning and text generation | — | 🧠 Buffett AI Agent (ai_languageModel) | 📌 Step 2 - AI Engine: 🧠 Step 2 - AI Engine — Agent tools (for qualitative research): 🔍 SerpAPI — moat, management, news, analyst opinions ; 🧮 Calculator — precise DCF arithmetic ; 📌 Setup Required: ⚙️ Required Setup — 1. Credentials to connect: 🤖 OpenAI API |
| 🔀 Check Agent Output | if (v2) | Validates that the agent output contains a closing HTML tag before delivery | 🧠 Buffett AI Agent | 📧 Send Gmail Report (true), ❌ Telegram Error Handler (false) | 📌 Step 3 - Output: 📤 Step 3 - Output — 1. IF node validates HTML output exists 2. ✅ Valid → Email + Telegram confirmation 3. ❌ Invalid → Error message to Telegram 📧 Gmail sends full HTML with inline CSS — renders as a styled financial report 💡 The verdict (BUY/HOLD/AVOID) is extracted from the HTML and shown in the Telegram reply and email subject. |
| 📧 Send Gmail Report | gmail (v2.1) | Sends the HTML analysis report via email with verdict in the subject | 🔀 Check Agent Output (true) | ✅ Telegram Confirmation | 📌 Step 3 - Output: 📤 Step 3 - Output — 1. IF node validates HTML output exists 2. ✅ Valid → Email + Telegram confirmation 3. ❌ Invalid → Error message to Telegram 📧 Gmail sends full HTML with inline CSS — renders as a styled financial report 💡 The verdict (BUY/HOLD/AVOID) is extracted from the HTML and shown in the Telegram reply and email subject. ; 📌 Email Config: ✉️ Email Recipient — Set your email in the 🔤 Extract Ticker & User node — look for the recipientEmail field. Change REPLACE_WITH_YOUR_GMAIL@gmail.com to your actual Gmail address. The Gmail account sending the report must be connected via OAuth2 in credentials. ; 📌 Setup Required: ⚙️ Required Setup — 1. Credentials to connect: 📧 Gmail OAuth2 |
| ✅ Telegram Confirmation | telegram (v1.2) | Sends a confirmation message to Telegram after email delivery with the extracted verdict | 📧 Send Gmail Report | — | 📌 Step 3 - Output: 📤 Step 3 - Output — 1. IF node validates HTML output exists 2. ✅ Valid → Email + Telegram confirmation 3. ❌ Invalid → Error message to Telegram 📧 Gmail sends full HTML with inline CSS — renders as a styled financial report 💡 The verdict (BUY/HOLD/AVOID) is extracted from the HTML and shown in the Telegram reply and email subject. |
| ❌ Telegram Error Handler | telegram (v1.2) | Sends an error message to Telegram when the agent output is invalid or the agent failed | 🔀 Check Agent Output (false) | — | 📌 Error Handling: ❌ Error Handling — Two error paths are covered: Path A — Invalid ticker (too short/long) → Caught before agent runs → Sends friendly Telegram hint ; Path B — Agent failure or bad output → continueOnFail keeps workflow alive → IF node checks for valid HTML → Routes to error message node |

---

### 4. Reproducing the Workflow from Scratch

**Prerequisites:**
- n8n instance (cloud or self-hosted) with the `@n8n/n8n-nodes-langchain` package installed (available in n8n ≥ 1.0 with AI nodes enabled)
- Four credential accounts: Telegram Bot, SerpAPI, OpenAI, Gmail

---

**Credential Setup (do this first):**

1. **Telegram Bot API**
   - Open Telegram, message `@BotFather`, use `/newbot`, follow prompts, copy the bot token.
   - In n8n, go to Credentials → Add → Telegram API → paste the token.

2. **SerpAPI**
   - Sign up at https://serpapi.com (free tier: 100 searches/month).
   - Copy your API key from the dashboard.
   - In n8n, go to Credentials → Add → SerpAPI → paste the key.

3. **OpenAI**
   - Sign up or log in at https://platform.openai.com.
   - Generate an API key with access to `gpt-4o-mini`.
   - In n8n, go to Credentials → Add → OpenAI API → paste the key.

4. **Gmail OAuth2**
   - In n8n, go to Credentials → Add → Gmail OAuth2.
   - Follow the OAuth2 consent flow to authorize n8n to send emails from your Gmail account.

---

**Node-by-Node Construction:**

5. **Create 📱 Telegram Trigger**
   - Add a **Telegram Trigger** node.
   - Set "Updates" to `message`.
   - Connect the Telegram Bot API credential.
   - Leave "Additional Fields" empty.
   - This becomes the workflow entry point.

6. **Create 🔤 Extract Ticker & User**
   - Add an **Edit Fields (Set)** node (type `n8n-nodes-base.set`, version 3.x).
   - Add four string assignments:
     - `ticker` = `{{ $json.message.text.toUpperCase().replace(/[^A-Z0-9.]/g, '').trim() }}`
     - `chat_id` = `{{ $json.message.chat.id }}`
     - `username` = `{{ $json.message.from.first_name || $json.message.from.username || 'Investor' }}`
     - `recipientEmail` = `your-actual-email@gmail.com` *(replace with your real Gmail address)*
   - Connect: Telegram Trigger → Extract Ticker & User.

7. **Create 🔎 Validate Ticker**
   - Add an **IF** node (type `n8n-nodes-base.if`, version 2).
   - Add a condition: **String** → **Matches Regex** → Left value: `={{ $json.ticker }}` → Right value (regex): `^[A-Z0-9.\-]{1,10}$`
   - Combinator: AND. Case-sensitive. Strict type validation.
   - Connect: Extract Ticker & User → Validate Ticker.

8. **Create ⚠️ Invalid Ticker Reply**
   - Add a **Telegram** node.
   - Connect the Telegram Bot API credential.
   - Text: A Markdown-formatted message explaining that the input is invalid and listing example tickers (AAPL, MSFT, GOOGL, AMZN, BRK.B).
   - Chat ID: `={{ $('🔤 Extract Ticker & User').item.json.chat_id }}`
   - Parse mode: Markdown.
   - Connect: Validate Ticker → (false branch) → Invalid Ticker Reply.

9. **Create 📨 Send Acknowledgement**
   - Add a **Telegram** node.
   - Connect the Telegram Bot API credential.
   - Text (dynamic Markdown): `🔍 *Analyzing {{ $('🔤 Extract Ticker & User').item.json.ticker }} using Warren Buffett's value investing framework...*` followed by a list of analysis steps and a note about 30–60 second wait time.
   - Chat ID: `={{ $('🔤 Extract Ticker & User').item.json.chat_id }}`
   - Parse mode: Markdown.
   - Connect: Validate Ticker → (true branch) → Send Acknowledgement.

10. **Create 🔍 SerpAPI Web Search (tool)**
    - Add a **SerpAPI** tool node (type `@n8n/n8n-nodes-langchain.toolSerpApi`).
    - Connect the SerpAPI credential.
    - Leave options at defaults (no extra parameters).
    - This node will be linked as an `ai_tool` to the Agent node later.

11. **Create 🧮 Calculator (DCF Math) (tool)**
    - Add a **Calculator** tool node (type `@n8n/n8n-nodes-langchain.toolCalculator`).
    - No parameters or credentials needed.
    - This node will be linked as an `ai_tool` to the Agent node later.

12. **Create 🤖 OpenAI GPT-4o-mini (language model)**
    - Add an **OpenAI Chat Model** node (type `@n8n/n8n-nodes-langchain.lmChatOpenAi`).
    - Connect the OpenAI API credential.
    - Set Model to `gpt-4o-mini` (select from the list).
    - Leave all other options at defaults. Do not enable built-in tools.
    - This node will be linked as `ai_languageModel` to the Agent node later.

13. **Create 🧠 Buffett AI Agent**
    - Add an **AI Agent** node (type `@n8n/n8n-nodes-langchain.agent`, version 3.1).
    - **Prompt Type:** Define (user-defined).
    - **Text:** `Analyze the company with ticker symbol: {{ $('🔤 Extract Ticker & User').item.json.ticker }}\n\nRequested by: {{ $('🔤 Extract Ticker & User').item.json.username }}\n\nDate of analysis: {{ $now.format('MMMM D, YYYY') }}`
    - **Max Iterations:** 15.
    - **System Message:** Paste the complete Buffett persona prompt (see Section 2, Block 3 for the full content). The prompt must include: persona definition, mandatory tool usage order (SerpAPI → Calculator → Final output), 10 research topics, the seven-section HTML template with all placeholder names, badge/verdict color rules, and the instruction to return only HTML with no markdown.
    - **Continue On Fail:** Enable (set to `true`).
    - Connect main input: Send Acknowledgement → Buffett AI Agent.
    - Connect ai_tool: SerpAPI Web Search → Buffett AI Agent (on the `ai_tool` input port).
    - Connect ai_tool: Calculator → Buffett AI Agent (on the `ai_tool` input port).
    - Connect ai_languageModel: OpenAI GPT-4o-mini → Buffett AI Agent (on the `ai_languageModel` input port).

14. **Create 🔀 Check Agent Output**
    - Add an **IF** node.
    - Add a condition: **String** → **Contains** → Left value: `={{ $json.output }}` → Right value: `</html>`
    - Combinator: AND. Case-sensitive. Strict type validation.
    - Connect: Buffett AI Agent → Check Agent Output.

15. **Create 📧 Send Gmail Report**
    - Add a **Gmail** node (type `n8n-nodes-base.gmail`, version 2.1).
    - Connect the Gmail OAuth2 credential.
    - **Send To:** `={{ $('🔤 Extract Ticker & User').item.json.recipientEmail }}`
    - **Subject:** `=📊 Buffett Analysis: {{ $('🔤 Extract Ticker & User').item.json.ticker }} — {{ $('🧠 Buffett AI Agent').item.json.output.match(/VERDICT: (BUY|HOLD|AVOID)/)?.[1] || 'Analysis Complete' }} | {{ $now.format('dd/LL/yyyy') }}`
    - **Message:** `={{ $('🧠 Buffett AI Agent').item.json.output }}`
    - **Options → Append Attribution:** Disabled (set to `false`).
    - Connect: Check Agent Output → (true branch) → Send Gmail Report.

16. **Create ✅ Telegram Confirmation**
    - Add a **Telegram** node.
    - Connect the Telegram Bot API credential.
    - Text (dynamic Markdown): `✅ *{{ $('🔤 Extract Ticker & User').item.json.ticker }} Analysis Complete!*` + verdict extraction + instructions to send another ticker.
    - Chat ID: `={{ $('🔤 Extract Ticker & User').item.json.chat_id }}`
    - Parse mode: Markdown.
    - Connect: Send Gmail Report → Telegram Confirmation.

17. **Create ❌ Telegram Error Handler**
    - Add a **Telegram** node.
    - Connect the Telegram Bot API credential.
    - Text (dynamic Markdown): `❌ *Analysis Failed*` + ticker reference + troubleshooting steps + truncated output: `{{ $json.output?.substring(0, 200) || 'No details available. Please try again.' }}`
    - Chat ID: `={{ $('🔤 Extract Ticker & User').item.json.chat_id }}`
    - Parse mode: Markdown.
    - Connect: Check Agent Output → (false branch) → Telegram Error Handler.

18. **Add Sticky Notes (optional but recommended for documentation)**
    - 📌 **Overview** — Top of canvas, describing the workflow purpose.
    - 📌 **Setup Required** — Left side, listing all four credentials and the email configuration step.
    - 📌 **Step 1 - Input** — Below the input nodes, explaining the input flow.
    - 📌 **Step 2 - AI Engine** — Below the agent and tools, describing the research tools.
    - 📌 **Step 3 - Output** — Right side, explaining the output validation and delivery.
    - 📌 **Error Handling** — Near the error nodes, describing both error paths.
    - 📌 **Email Config** — Near the Gmail node, reminding about the recipientEmail field.

19. **Activate the Workflow**
    - Save the workflow.
    - Toggle the workflow to **Active**.
    - Verify the Telegram webhook is registered (n8n does this automatically when the workflow is activated).
    - Send a test ticker (e.g. `AAPL`) to your Telegram bot and confirm you receive the acknowledgement, the email, and the confirmation.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SerpAPI free tier provides 100 searches per month — sufficient for light personal use but may be exhausted quickly if many analyses are run in one day. | https://serpapi.com |
| The `recipientEmail` field in the Extract Ticker & User node ships with a placeholder value `user@example.com`. This must be changed to a real Gmail address before the workflow can deliver reports. | Node: 🔤 Extract Ticker & User |
| The AI Agent's system prompt is approximately 3000+ words. GPT-4o-mini has a 128K context window but may occasionally truncate very long HTML outputs. If the output is truncated, the `</html>` check in Check Agent Output will route to the error handler. | OpenAI model documentation: https://platform.openai.com/docs/models |
| The Gmail node sends the HTML as the email body (not as an attachment). Gmail clips messages over approximately 102 KB. The Buffett HTML template is designed for inline CSS for maximum email client compatibility. | Gmail clipping behavior: https://support.google.com/mail/answer/6594 |
| The workflow uses `continueOnFail: true` on the AI Agent node. This means if SerpAPI or OpenAI returns an error, the workflow does not stop — instead it passes the error information forward and the Check Agent Output IF node catches it. This is intentional but means error details may appear in the `output` field. | Node: 🧠 Buffett AI Agent |
| The verdict extraction regex `VERDICT: (BUY|HOLD|AVOID)` is used in both the Gmail subject line and the Telegram confirmation. If the AI deviates from the exact HTML template format (different spacing, lowercase, or extra HTML tags), the regex will fail and the fallback text ("Analysis Complete" or "See email for details") will be used instead. | Nodes: 📧 Send Gmail Report, ✅ Telegram Confirmation |
| The Telegram Markdown parse mode used across all Telegram nodes is the v1 legacy format. Underscores in text may cause unintended italic formatting. If you encounter parse errors, consider switching to `MarkdownV2` or `HTML` parse mode and adjusting the message templates. | Telegram Bot API formatting: https://core.telegram.org/bots/api#formatting-options |
| This workflow is tagged with **Finance**, **Telegram**, and **AI Agent** in n8n for organizational purposes. | — |
| The workflow is currently set to **inactive** (`active: false`) and must be manually activated after deployment. | — |