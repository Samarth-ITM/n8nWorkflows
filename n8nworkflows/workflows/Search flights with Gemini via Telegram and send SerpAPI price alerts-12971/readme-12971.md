Search flights with Gemini via Telegram and send SerpAPI price alerts

https://n8nworkflows.xyz/workflows/search-flights-with-gemini-via-telegram-and-send-serpapi-price-alerts-12971


# Search flights with Gemini via Telegram and send SerpAPI price alerts

## 1. Workflow Overview

**Purpose:** This workflow combines (1) scheduled flight price monitoring with threshold-based Telegram alerts and (2) an AI flight assistant on Telegram powered by Google Gemini that can call SerpAPI Google Flights tools (one-way / round-trip) and respond in Telegram HTML.

**Target use cases**
- Passive monitoring of a specific route (e.g., HAN → BKK) and alerting when the cheapest fare is below a configured threshold.
- Interactive flight search in natural language via Telegram, with conversation context and tool-based retrieval (SerpAPI).

### 1.1 Block A — Documentation / Notes Layer
Sticky notes that describe setup, usage, and optimization recommendations.

### 1.2 Block B — Workflow 1: Automated Price Monitoring (Scheduled)
Schedule trigger → set route + threshold + chat ID → SerpAPI Google Flights search (one-way for today) → extract cheapest fare → filter by threshold → send Telegram alert.

### 1.3 Block C — Workflow 2: AI Flight Assistant (Telegram + Gemini + Tools)
Telegram trigger → LangChain AI Agent (Gemini) with memory → agent may invoke SerpAPI “tool” nodes for flight search → agent returns formatted HTML → send response to Telegram.

---

## 2. Block-by-Block Analysis

### Block A — Documentation / Notes Layer
**Overview:** Provides human guidance for configuration, expected behavior, and optimization. These nodes do not execute logic.

**Nodes involved:**
- 📌 Workflow Introduction
- 📊 Workflow 1 Documentation
- 💬 Workflow 2 Documentation
- 📈 Performance & Optimization
- Sticky Note

#### Node: 📌 Workflow Introduction
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Large note containing overview, requirements, and author links.
- **Connections:** None.
- **Edge cases:** None (non-executing).
- **Notable links:**  
  - https://serpapi.com  
  - https://aistudio.google.com/app/apikey  
  - https://nguyenthieutoan.com  
  - https://n8n.io/creators/nguyenthieutoan

#### Node: 📊 Workflow 1 Documentation
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Describes scheduled monitoring steps and example alert format.
- **Connections:** None.

#### Node: 💬 Workflow 2 Documentation
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Describes AI assistant behavior, memory, and Telegram HTML formatting.
- **Connections:** None.

#### Node: 📈 Performance & Optimization
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Performance estimates and recommendations (caching, retries, rate limiting).
- **Connections:** None.

#### Node: Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** “EDIT THIS NODES” reminder, visually placed near the scheduled monitoring section.
- **Connections:** None.

---

### Block B — Workflow 1: Automated Price Monitoring (Scheduled)
**Overview:** On a fixed schedule, the workflow searches Google Flights via SerpAPI for today’s one-way fares on a configured route, extracts the cheapest price, compares it to a threshold, and sends a Telegram alert if it’s below the threshold.

**Nodes involved:**
- ⏰ Schedule Trigger
- ⚙️ Edit Fields
- ✈️ Google Flights Search
- 💰 Extract Best Price
- 🔍 Filter by Price
- 📱 Send Alert

#### Node: ⏰ Schedule Trigger
- **Type / role:** Schedule Trigger (entry point for monitoring)
- **Configuration choices:** Runs every **7 days at 07:30**.
- **Output:** Triggers one execution with an empty/default item.
- **Connections:** ⏰ Schedule Trigger → ⚙️ Edit Fields
- **Edge cases / failures:**
  - If workflow is inactive, no runs occur.
  - Timezone behavior depends on n8n instance settings.

#### Node: ⚙️ Edit Fields
- **Type / role:** Set node (configuration/constants)
- **Configuration choices:** Hard-codes monitoring parameters:
  - `Departure` = `"HAN"`
  - `Destination` = `"BKK"`
  - `Acceptable Price (USD)` = `1000`
  - `TelegramID` = `"YOUR_TELEGRAM_ID"`
- **Connections:** ⚙️ Edit Fields → ✈️ Google Flights Search
- **Key risk (important):** The later Filter node references a **different field name** (`Acceptable Price (USD))` with an extra `)`), which will likely evaluate to `undefined` and break/disable correct filtering.
- **Edge cases / failures:**
  - Wrong IATA codes → SerpAPI may return empty/no flight data.
  - TelegramID left as placeholder → message send fails or goes to wrong chat.

#### Node: ✈️ Google Flights Search
- **Type / role:** SerpAPI node (API call to Google Flights)
- **Configuration choices:**
  - Operation: `google_flights`
  - One-way (`type = 2`)
  - `departure_id` = expression from `Departure`
  - `arrival_id` = expression from `Destination`
  - `outbound_date` = `{{ $now.toFormat('yyyy-MM-dd') }}` (today)
  - Additional fields: `gl=vn`, `hl=en`, `currency=USD`
- **Credentials:** SerpAPI credential required.
- **Connections:** ✈️ Google Flights Search → 💰 Extract Best Price
- **Edge cases / failures:**
  - SerpAPI auth/key quota exceeded (HTTP error).
  - Google Flights may not return `best_flights` / `other_flights` for some routes/dates.
  - Currency mismatch vs notes: notes mention VND in places, but node uses USD.

#### Node: 💰 Extract Best Price
- **Type / role:** Code node (normalize + compute cheapest)
- **Configuration choices (logic):**
  1. Reads `$input.first().json` and `search_metadata`
  2. Merges `best_flights` + `other_flights`
  3. Extracts numeric price by stripping non-digits
  4. Extracts airline + departure/arrival times from `flights[0]`
  5. Sorts by price ascending and selects cheapest
  6. Outputs:
     - `ok` (boolean)
     - `minPrice` (number)
     - `message` like:  
       `✈️ Cheapest Ticket\nPrice: 1,234 USD\nAirline: ...\nTime: ... → ...`
     - `meta`
- **Connections:** 💰 Extract Best Price → 🔍 Filter by Price
- **Edge cases / failures:**
  - If no flights: returns `ok:false` and message “No flight data available” (but still passes downstream unless filtered elsewhere).
  - Price parsing strips all non-digits; this can misread decimals or formatted strings (usually acceptable for integer-like fares).
  - If SerpAPI schema changes, paths like `f.flights?.[0]?.departure_airport?.time` may be missing.

#### Node: 🔍 Filter by Price
- **Type / role:** Filter node (conditional gate)
- **Configuration choices:** Condition: `{{$json.minPrice}}` **lt** `{{ $('⚙️ Edit Fields').item.json["Acceptable Price (USD))"] }}`
- **Connections:** On pass → 📱 Send Alert; on fail → stops.
- **Major issue:** The right-side field name appears incorrect (`Acceptable Price (USD))`), while the Set node creates `Acceptable Price (USD)`. This likely results in:
  - Right value becomes `undefined` → comparison may always fail or behave unexpectedly depending on n8n coercion.
- **Edge cases / failures:**
  - If `minPrice` missing because `ok:false`, comparison may fail.
  - Loose type validation is enabled; unexpected coercions may occur.

#### Node: 📱 Send Alert
- **Type / role:** Telegram node (send message)
- **Configuration choices:**
  - `chatId` = `{{ $('⚙️ Edit Fields').item.json.TelegramID }}`
  - `text` = `{{ $json.message }}`
- **Credentials:** Telegram bot credential required.
- **Connections:** Terminal node in monitoring path.
- **Edge cases / failures:**
  - Chat ID invalid or bot not started by user → Telegram API error (403 / “chat not found”).
  - Message content length/format issues are minimal here (plain text).

---

### Block C — Workflow 2: AI Flight Assistant (Telegram + Gemini + Tools)
**Overview:** When a Telegram user sends a message, an AI Agent (Gemini) processes it. The agent uses memory keyed by Telegram user ID and can call SerpAPI tool nodes to retrieve flights. The agent returns a Telegram-HTML formatted reply, sent back to the user.

**Nodes involved:**
- 💬 Telegram Trigger
- 🤖 AI Agent1
- 🧠 Google Gemini Model1
- 🧠 Conversation Memory1
- 🔄 Round Trip Search1
- ➡️ One-Way Search1
- 📨 Send Response

#### Node: 💬 Telegram Trigger
- **Type / role:** Telegram Trigger (entry point for assistant)
- **Configuration choices:** Listens to `message` updates.
- **Credentials:** Telegram bot credential required.
- **Output:** Typical Telegram update payload, used later for:
  - user text: `$json.message.text`
  - user id: `$json.message.from.id`
- **Connections:** 💬 Telegram Trigger → 🤖 AI Agent1
- **Edge cases / failures:**
  - Bot privacy settings / group permissions can prevent messages.
  - Telegram webhook configuration handled by n8n; invalid public URL can break delivery.

#### Node: 🤖 AI Agent1
- **Type / role:** LangChain Agent node (tool-using conversational agent)
- **Configuration choices:**
  - Input text: `={{ $json.message.text }}`
  - System prompt enforces:
    - Must be concise, direct, efficient
    - Must not “scan cheapest day of month” without a specific date
    - Tool usage rules: only call tools when **origin, destination, departure date** are present; for round-trip missing return date, default `+5 days` and disclose it
    - Output must be **Telegram HTML** with allowed tags
- **Inputs (special connections):**
  - Language model from 🧠 Google Gemini Model1 (ai_languageModel)
  - Memory from 🧠 Conversation Memory1 (ai_memory)
  - Tools from 🔄 Round Trip Search1 and ➡️ One-Way Search1 (ai_tool)
- **Outputs:** Produces an object that includes `output` used by 📨 Send Response.
- **Connections:** 🤖 AI Agent1 → 📨 Send Response
- **Edge cases / failures:**
  - Model/API auth failures, rate limits, timeouts.
  - Agent may request parameters not provided; in that case it should ask follow-up questions (per system prompt).
  - Telegram HTML: if agent outputs unsupported tags, Telegram may reject or strip formatting.

#### Node: 🧠 Google Gemini Model1
- **Type / role:** Google Gemini chat model connector (LLM backend)
- **Configuration choices:** Uses a Gemini model (noted as `gemini-2.0-flash-exp` in node notes).
- **Credentials:** Google AI Studio / PaLM (Gemini) API key required.
- **Connections:** 🧠 Google Gemini Model1 → 🤖 AI Agent1 (ai_languageModel)
- **Edge cases / failures:**
  - Wrong API key / disabled API / quota exceeded.
  - Model name availability depends on region/account and n8n node version.

#### Node: 🧠 Conversation Memory1
- **Type / role:** Buffer window memory (context retention)
- **Configuration choices:**
  - `sessionKey` = `={{ $json.message.from.id }}`
  - `sessionIdType` = custom key
  - Keeps a recent window of messages (exact window size not specified in parameters shown).
- **Connections:** 🧠 Conversation Memory1 → 🤖 AI Agent1 (ai_memory)
- **Edge cases / failures:**
  - If Telegram payload changes or `message.from.id` missing (rare), session key expression fails.
  - Memory growth/retention depends on node defaults and n8n storage.

#### Node: 🔄 Round Trip Search1
- **Type / role:** SerpAPI Tool node (callable by agent)
- **Configuration choices:**
  - Operation: `google_flights`
  - Parameters are provided via `$fromAI(...)` (agent-filled):
    - `departure_id`, `arrival_id`, `outbound_date`, `return_date`
  - Additional fields: `gl=vn`, `hl=en`, `currency=USD`
- **Credentials:** SerpAPI credential required.
- **Connections:** 🔄 Round Trip Search1 → 🤖 AI Agent1 (ai_tool)
- **Edge cases / failures:**
  - If agent provides invalid date format, SerpAPI returns errors/empty results.
  - Same SerpAPI quota/auth risks as monitoring workflow.

#### Node: ➡️ One-Way Search1
- **Type / role:** SerpAPI Tool node (callable by agent)
- **Configuration choices:**
  - One-way (`type=2`)
  - Agent supplies: `departure_id`, `arrival_id`, `outbound_date` via `$fromAI(...)`
  - Additional fields: `gl=vn`, `hl=en`, `currency=USD`
- **Credentials:** SerpAPI credential required.
- **Connections:** ➡️ One-Way Search1 → 🤖 AI Agent1 (ai_tool)
- **Edge cases / failures:** Same as round-trip tool.

#### Node: 📨 Send Response
- **Type / role:** Telegram node (send assistant reply)
- **Configuration choices:**
  - `chatId` = `={{ $('💬 Telegram Trigger').item.json.message.from.id }}`
  - `text` = `={{ $json.output }}`
  - `parse_mode` = `HTML`
  - `appendAttribution` = false
- **Credentials:** Telegram bot credential required.
- **Connections:** Terminal node for assistant path.
- **Edge cases / failures:**
  - If agent output includes invalid HTML tags/entities → Telegram may error.
  - Large messages may exceed Telegram limits (needs truncation/summary if that occurs).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📌 Workflow Introduction | n8n-nodes-base.stickyNote | Documentation / overview |  |  | # 🎯 Smart Flight Price Alert Bot with AI Assistant… (includes links: https://serpapi.com, https://aistudio.google.com/app/apikey, https://nguyenthieutoan.com, https://n8n.io/creators/nguyenthieutoan) |
| 📊 Workflow 1 Documentation | n8n-nodes-base.stickyNote | Documentation for scheduled monitoring |  |  | ## 🔄 Workflow 1: Automated Price Monitoring… |
| 💬 Workflow 2 Documentation | n8n-nodes-base.stickyNote | Documentation for AI assistant |  |  | ## 🤖 Workflow 2: AI Flight Assistant… |
| ⏰ Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts scheduled monitoring |  | ⚙️ Edit Fields | ## EDIT THIS NODES |
| ⚙️ Edit Fields | n8n-nodes-base.set | Configure route/threshold/chat for monitoring | ⏰ Schedule Trigger | ✈️ Google Flights Search | ## EDIT THIS NODES |
| ✈️ Google Flights Search | n8n-nodes-serpapi.serpApi | SerpAPI Google Flights API call (monitoring) | ⚙️ Edit Fields | 💰 Extract Best Price |  |
| 💰 Extract Best Price | n8n-nodes-base.code | Parse results, find cheapest, format message | ✈️ Google Flights Search | 🔍 Filter by Price |  |
| 🔍 Filter by Price | n8n-nodes-base.filter | Gate: cheapest price below threshold | 💰 Extract Best Price | 📱 Send Alert |  |
| 📱 Send Alert | n8n-nodes-base.telegram | Send monitoring alert to Telegram | 🔍 Filter by Price |  |  |
| 💬 Telegram Trigger | n8n-nodes-base.telegramTrigger | Starts AI assistant from incoming chat |  | 🤖 AI Agent1 |  |
| 🤖 AI Agent1 | @n8n/n8n-nodes-langchain.agent | LLM agent orchestrating tools + formatting | 💬 Telegram Trigger; (ai_model, tools, memory) | 📨 Send Response |  |
| 🧠 Google Gemini Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backend for agent |  | 🤖 AI Agent1 |  |
| 🔄 Round Trip Search1 | n8n-nodes-serpapi.serpApiTool | Agent tool: round-trip flight search |  | 🤖 AI Agent1 |  |
| ➡️ One-Way Search1 | n8n-nodes-serpapi.serpApiTool | Agent tool: one-way flight search |  | 🤖 AI Agent1 |  |
| 🧠 Conversation Memory1 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Per-user short-term memory |  | 🤖 AI Agent1 |  |
| 📨 Send Response | n8n-nodes-base.telegram | Send AI HTML response back to user | 🤖 AI Agent1 |  |  |
| 📈 Performance & Optimization | n8n-nodes-base.stickyNote | Non-executing optimization notes |  |  | ## 📊 Workflow Statistics & Performance… |
| Sticky Note | n8n-nodes-base.stickyNote | Non-executing reminder |  |  | ## EDIT THIS NODES |

---

## 4. Reproducing the Workflow from Scratch

### A) Prerequisites / Credentials
1. **Telegram Bot**
   - Create bot via **@BotFather** → get bot token.
   - In n8n, create **Telegram API** credentials using the token.
2. **SerpAPI**
   - Create account and API key at https://serpapi.com
   - In n8n, create **SerpAPI** credentials with the key.
3. **Google Gemini (AI Studio)**
   - Create API key at https://aistudio.google.com/app/apikey
   - In n8n, create **Google Gemini / PaLM** credentials (as required by the Gemini node).

---

### B) Workflow 1 — Automated Price Monitoring
1. Add **Schedule Trigger**
   - Interval: every **7 days**
   - Trigger time: **07:30**
2. Add **Set** node named **⚙️ Edit Fields**
   - Add fields:
     - `Departure` (string) e.g. `HAN`
     - `Destination` (string) e.g. `BKK`
     - `Acceptable Price (USD)` (number) e.g. `1000`
     - `TelegramID` (string) your numeric Telegram user/chat id
3. Connect: **⏰ Schedule Trigger → ⚙️ Edit Fields**
4. Add **SerpAPI** node named **✈️ Google Flights Search**
   - Operation: **Google Flights**
   - One-way: set `type = 2`
   - `departure_id` = `{{ $json.Departure }}`
   - `arrival_id` = `{{ $json.Destination }}`
   - `outbound_date` = `{{ $now.toFormat('yyyy-MM-dd') }}`
   - Additional: `gl=vn`, `hl=en`, `currency=USD`
   - Select your **SerpAPI credentials**
5. Connect: **⚙️ Edit Fields → ✈️ Google Flights Search**
6. Add **Code** node named **💰 Extract Best Price**
   - Paste logic that merges `best_flights` + `other_flights`, parses numeric price, sorts, outputs `minPrice` and `message`.
7. Connect: **✈️ Google Flights Search → 💰 Extract Best Price**
8. Add **Filter** node named **🔍 Filter by Price**
   - Condition: `{{ $json.minPrice }}` **is less than** `{{ $('⚙️ Edit Fields').item.json["Acceptable Price (USD)"] }}`
   - (Ensure the field name matches exactly; do **not** include an extra `)`.)
9. Connect: **💰 Extract Best Price → 🔍 Filter by Price**
10. Add **Telegram** node named **📱 Send Alert**
   - Operation: send message
   - `chatId` = `{{ $('⚙️ Edit Fields').item.json.TelegramID }}`
   - `text` = `{{ $json.message }}`
   - Select your **Telegram API credentials**
11. Connect: **🔍 Filter by Price (true output) → 📱 Send Alert**
12. Activate workflow.

---

### C) Workflow 2 — AI Flight Assistant (Telegram + Gemini + SerpAPI tools)
1. Add **Telegram Trigger** node named **💬 Telegram Trigger**
   - Updates: `message`
   - Select your **Telegram API credentials**
2. Add **LangChain AI Agent** node named **🤖 AI Agent1**
   - Input text: `{{ $json.message.text }}`
   - Prompt type: “define” (system message)
   - Paste a system message that:
     - requires origin/destination/date before tool calls
     - supports round-trip default return date = departure + 5 days
     - enforces Telegram HTML output
3. Add **Google Gemini Chat Model** node named **🧠 Google Gemini Model1**
   - Choose the Gemini model available in your node/version (e.g., Flash-tier)
   - Select your **Gemini credentials**
4. Connect (AI model): **🧠 Google Gemini Model1 → 🤖 AI Agent1** via **ai_languageModel**
5. Add **Memory Buffer Window** node named **🧠 Conversation Memory1**
   - Session key: `{{ $json.message.from.id }}`
   - Session id type: custom key
6. Connect (memory): **🧠 Conversation Memory1 → 🤖 AI Agent1** via **ai_memory**
7. Add **SerpAPI Tool** node named **➡️ One-Way Search1**
   - Operation: Google Flights, set `type=2`
   - Parameters should be AI-filled (use the node’s “from AI” parameter helpers) for:
     - `departure_id`
     - `arrival_id`
     - `outbound_date` (yyyy-MM-dd)
   - Additional: `gl=vn`, `hl=en`, `currency=USD`
   - SerpAPI credentials
8. Add **SerpAPI Tool** node named **🔄 Round Trip Search1**
   - Operation: Google Flights
   - Parameters AI-filled for:
     - `departure_id`
     - `arrival_id`
     - `outbound_date`
     - `return_date`
   - Additional: `gl=vn`, `hl=en`, `currency=USD`
   - SerpAPI credentials
9. Connect both tools to agent via **ai_tool**:
   - **➡️ One-Way Search1 → 🤖 AI Agent1** (ai_tool)
   - **🔄 Round Trip Search1 → 🤖 AI Agent1** (ai_tool)
10. Connect trigger to agent:
   - **💬 Telegram Trigger → 🤖 AI Agent1**
11. Add **Telegram** node named **📨 Send Response**
   - `chatId` = `{{ $('💬 Telegram Trigger').item.json.message.from.id }}`
   - `text` = `{{ $json.output }}`
   - `parse_mode` = `HTML`
   - `appendAttribution` = false
   - Telegram credentials
12. Connect: **🤖 AI Agent1 → 📨 Send Response**
13. Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SerpAPI key and pricing/free tier referenced | https://serpapi.com |
| Google Gemini API key setup | https://aistudio.google.com/app/apikey |
| Template author | https://nguyenthieutoan.com |
| More templates by author | https://n8n.io/creators/nguyenthieutoan |
| Telegram user ID retrieval hint: message @userinfobot | Mentioned in ⚙️ Edit Fields notes |
| Known configuration inconsistency to fix: Filter node references wrong threshold field name | Change to `Acceptable Price (USD)` exactly |
| Currency consistency note: some notes mention VND but nodes use USD | Align `currency` fields and message text as desired |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.