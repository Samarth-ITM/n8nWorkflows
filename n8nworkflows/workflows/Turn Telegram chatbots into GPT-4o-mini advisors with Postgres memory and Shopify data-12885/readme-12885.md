Turn Telegram chatbots into GPT-4o-mini advisors with Postgres memory and Shopify data

https://n8nworkflows.xyz/workflows/turn-telegram-chatbots-into-gpt-4o-mini-advisors-with-postgres-memory-and-shopify-data-12885


# Turn Telegram chatbots into GPT-4o-mini advisors with Postgres memory and Shopify data

## 1. Workflow Overview

**Purpose:**  
This workflow turns a Telegram chatbot into a **GPT-4o-mini powered advisor** (“SOFÍA”, a coffee barista persona) that answers using:
- **Persistent conversation memory** stored in **Postgres** (per Telegram chat/user),
- **Real-time Shopify product catalog** (names + prices) to avoid hallucinations,
- A strict **JSON response format**, validated and repaired with a fallback.

**Target use cases:**
- Customer support / sales assistant that remembers user preferences over time
- Product recommendation based on live catalog data
- Multi-turn conversations where identity (user name) is required before advising

### 1.1 Input Reception & User Identification
Receives Telegram messages and extracts a stable `chat_id` + message text for downstream nodes and memory session keying.

### 1.2 Real-Time Business Context (Shopify)
Loads the current Shopify product list so the AI can quote exact product titles and prices.

### 1.3 AI Reasoning with Memory (OpenAI + LangChain Agent + Postgres Memory)
Runs an AI Agent using GPT-4o-mini, injecting:
- user message,
- Postgres chat history,
- Shopify product context inside the system prompt.

### 1.4 Output Validation & Delivery (JSON parsing + Telegram reply)
Parses/repairs the agent output into valid JSON and sends `response` back to Telegram, with a safe fallback if parsing fails.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & User Identification
**Overview:** Captures incoming Telegram messages and normalizes the payload to a minimal structure (`text`, `chat_id`) used throughout the workflow (including memory session keying).

**Nodes involved:**
- `Telegram Trigger`
- `Code in JavaScript`

#### Node: Telegram Trigger
- **Type / role:** `n8n-nodes-base.telegramTrigger` — entry point; listens for Telegram updates.
- **Configuration (interpreted):**
  - Updates: `message` (only standard chat messages)
  - Uses Telegram credentials: **“Telegram account”**
- **Outputs:** Telegram update JSON (includes `message.text`, `message.chat.id`, etc.)
- **Connections:**
  - Output → `Code in JavaScript`
- **Potential failures / edge cases:**
  - Telegram credential invalid/revoked
  - Bot not added to chat / privacy settings
  - Non-text messages (photos, stickers) may not contain `message.text` → downstream expressions must handle empty string

#### Node: Code in JavaScript
- **Type / role:** `n8n-nodes-base.code` — normalizes incoming Telegram payload.
- **Key logic:**
  - Extracts:
    - `chatMessage = $json.message.text || ""`
    - `chatId = $json.message.chat.id || 0`
  - Returns a clean object: `{ text, chat_id }`
- **Why this matters:**
  - Creates a guaranteed `chat_id` for Postgres memory session keys.
- **Connections:**
  - Input: `Telegram Trigger`
  - Output → `Get many products`
- **Potential failures / edge cases:**
  - If Telegram payload differs (e.g., edited messages), `message` can be missing → would throw unless guarded. Current code assumes `$json.message` exists.
  - If `chat_id` becomes `0`, memory will collide for all such failures (bad). Consider defensive checks.

---

### Block 2 — Real-Time Context (Shopify)
**Overview:** Fetches the live product catalog from Shopify so the AI can list exact product names and prices.

**Nodes involved:**
- `Get many products`

#### Node: Get many products
- **Type / role:** `n8n-nodes-base.shopify` — reads product data from Shopify.
- **Configuration (interpreted):**
  - Resource: `product`
  - Operation: `getAll`
  - Auth: OAuth2
  - Credentials: **“Shopify Access Token account”**
- **Output:** An item per product (with fields like `title`, `variants[0].price`, etc.)
- **Connections:**
  - Input: `Code in JavaScript`
  - Output → `AI Agent`
- **Potential failures / edge cases:**
  - OAuth token expired/scopes missing (e.g., products read scope)
  - Large catalogs may paginate; “getAll” can be slow/time out depending on store size
  - Some products may have empty `variants` or missing `variants[0].price` → the system prompt expression can break or produce `undefined`

---

### Block 3 — AI Reasoning & Memory
**Overview:** Runs the AI Agent with a strict system protocol (name capture, catalog precision rules, memory rules). It uses Postgres chat history to remember the user and preferences.

**Nodes involved:**
- `OpenAI Chat Model`
- `Postgres Chat Memory`
- `AI Agent`

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM for the agent.
- **Configuration (interpreted):**
  - Model: `gpt-4o-mini`
  - Credentials: **“OpenAi account”**
  - No additional tools enabled (built-in tools empty)
- **Connections:**
  - Output (AI language model channel) → `AI Agent`
- **Potential failures / edge cases:**
  - API key invalid / billing / rate limits
  - Model unavailable in region/account
  - Timeouts on long contexts (especially if many products + memory)

#### Node: Postgres Chat Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryPostgresChat` — persistent chat memory backend.
- **Configuration (interpreted):**
  - Session ID type: `customKey`
  - Session key: `{{ $('Code in JavaScript').item.json.chat_id }}`
  - Context window length: `30` (last 30 messages/interactions retained in context)
  - Credentials: **“Postgres account 2”**
- **Connections:**
  - Output (AI memory channel) → `AI Agent`
- **Potential failures / edge cases:**
  - Postgres connectivity/auth/SSL issues
  - Missing required tables/schema (node may auto-create depending on version/config)
  - If `chat_id` is wrong/empty, users may share memory or lose history

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + memory + model to produce the response.
- **Configuration (interpreted):**
  - User text input: `{{ $('Telegram Trigger').item.json.message.text }}`
    - Note: This bypasses the normalized `text` from `Code in JavaScript` (works for text messages, but less robust for non-text updates).
  - **System message** (Spanish) defines:
    - Persona: “SOFÍA” (Zyntha Coffee barista)
    - Mandatory greeting protocol depending on whether the user’s name is known from memory
    - “Catalog precision” rule: must list exact product names and prices from Shopify section
    - “Focus rule” for discussing a specific coffee
    - Strict JSON output schema:
      ```json
      {
        "intent": "setup | recommendation | question",
        "response": "...",
        "product": "Name or null",
        "qty": null
      }
      ```
    - Injected product list (dynamic expression):
      - `{{ $('Get many products').all().map(p => p.json.title + " (Precio: $" + p.json.variants[0].price + ")").join(', ') }}`
  - Output parser enabled (`hasOutputParser: true`) — expects structured output.
- **Connections:**
  - Inputs:
    - Model from `OpenAI Chat Model`
    - Memory from `Postgres Chat Memory`
    - Main input from `Get many products` (this is how the agent “sees” the Shopify items in the execution context)
  - Output → `Code in JavaScript 2`
- **Potential failures / edge cases:**
  - If Shopify products output is large, system prompt becomes very long → higher latency / context limits
  - If `variants[0]` missing, the system message expression may error or render invalid text
  - If the agent outputs non-JSON (or JSON with extra text), downstream parsing may fail (handled by fallback)

---

### Block 4 — Output Validation & Delivery
**Overview:** Ensures the agent response is valid JSON and always returns a safe message if parsing fails, then replies to the Telegram user.

**Nodes involved:**
- `Code in JavaScript 2`
- `Send a text message`

#### Node: Code in JavaScript 2
- **Type / role:** `n8n-nodes-base.code` — attempts to extract and parse JSON from the agent output; otherwise returns a fallback JSON object.
- **Key logic (interpreted):**
  - Reads `items[0].json.output` (agent raw output).
  - Extracts substring between first `{` and last `}` to tolerate extra text.
  - `JSON.parse()` that substring.
  - **Fallback:** if parsing fails, returns a forced `"setup"` response asking for the user’s name.
- **Important implementation detail in n8n:**
  - The code returns a plain object (`return parsed;` / `return { ... }`) instead of `return [{ json: parsed }]`. In many n8n versions, the Code node should return an array of items. If your n8n build is strict, you must wrap outputs as items.
- **Connections:**
  - Input: `AI Agent`
  - Output → `Send a text message`
- **Potential failures / edge cases:**
  - `items[0].json.output` missing (agent node output shape changes) → runtime error
  - If AI returns JSON but with trailing commas/unquoted keys → parse fails → fallback triggers
  - If fallback triggers too often, users will be stuck in “tell me your name” loop

#### Node: Send a text message
- **Type / role:** `n8n-nodes-base.telegram` — sends the final response back to Telegram.
- **Configuration (interpreted):**
  - Chat ID: `{{ $('Code in JavaScript').first().json.chat_id }}`
  - Text: `{{ $json.response }}`
  - Credentials: **“Telegram account”**
- **Connections:**
  - Input: `Code in JavaScript 2`
  - Output: none (end)
- **Potential failures / edge cases:**
  - If `chat_id` extraction failed earlier, message may go to chat `0` (invalid) and fail
  - Telegram message length limits (very long AI responses can fail)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Entry point: receives Telegram messages | — | Code in JavaScript | ## Section 1 — Input & User Identification; Receives the incoming message and extracts the chat_id used to uniquely identify the user. This ensures every message can be linked to the correct conversation history. |
| Code in JavaScript | n8n-nodes-base.code | Normalize payload; extract `text` and `chat_id` | Telegram Trigger | Get many products | ## Section 1 — Input & User Identification; Receives the incoming message and extracts the chat_id used to uniquely identify the user. This ensures every message can be linked to the correct conversation history. |
| Get many products | n8n-nodes-base.shopify | Fetch live Shopify product catalog | Code in JavaScript | AI Agent | ## Section 2 — Real-Time Context; Fetches the live product catalog directly from the store. This prevents the AI from hallucinating product names or prices. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider (gpt-4o-mini) | — (model input implicit) | AI Agent | ## Section 3 — AI Reasoning & Memory; Combines user input, historical memory, and real-time data to generate a contextual response. This is where personalization, rules, and long-term memory are applied. |
| Postgres Chat Memory | @n8n/n8n-nodes-langchain.memoryPostgresChat | Persistent chat memory keyed by `chat_id` | — (memory input implicit) | AI Agent | ## Section 3 — AI Reasoning & Memory; Combines user input, historical memory, and real-time data to generate a contextual response. This is where personalization, rules, and long-term memory are applied. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates prompt + memory + model; generates JSON output | Get many products (+ model + memory channels) | Code in JavaScript 2 | ## Section 3 — AI Reasoning & Memory; Combines user input, historical memory, and real-time data to generate a contextual response. This is where personalization, rules, and long-term memory are applied. |
| Code in JavaScript 2 | n8n-nodes-base.code | Parse/repair JSON; fallback response | AI Agent | Send a text message | ## Section 4 — Output & Delivery; Validates that the AI response is valid JSON and applies a fallback if needed. Sends the final, safe response back to the user via Telegram. / ⚠️ Important; This workflow requires valid JSON output from the AI. If you modify the system prompt, make sure the response format is preserved to avoid parsing errors. |
| Send a text message | n8n-nodes-base.telegram | Sends final response to Telegram | Code in JavaScript 2 | — | ## Section 4 — Output & Delivery; Validates that the AI response is valid JSON and applies a fallback if needed. Sends the final, safe response back to the user via Telegram. / ⚠️ Important; This workflow requires valid JSON output from the AI. If you modify the system prompt, make sure the response format is preserved to avoid parsing errors. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## AI Advisor with Memory and Real-Time Context; How it works… How to set up… Customization… |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## Section 1 — Input & User Identification; Receives the incoming message… |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## Section 2 — Real-Time Context; Fetches the live product catalog… |
| Sticky Note10 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## Section 3 — AI Reasoning & Memory; Combines user input… |
| Sticky Note11 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## Section 4 — Output & Delivery; Validates that the AI response… |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ⚠️ Important; This workflow requires valid JSON output from the AI… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “Telegram Trigger”** (`Telegram Trigger` node)
   - Updates: `message`
   - Add Telegram credentials (Bot Token) under a credential like **Telegram account**.

2. **Create “Code in JavaScript”** (`Code` node)
   - Paste logic to extract:
     - `text` from `message.text` (default empty string)
     - `chat_id` from `message.chat.id` (default 0)
   - Connect: **Telegram Trigger → Code in JavaScript**

3. **Create “Get many products”** (`Shopify` node)
   - Resource: `Product`
   - Operation: `Get All`
   - Authentication: OAuth2
   - Connect Shopify OAuth2 credentials (store + scopes for reading products)
   - Connect: **Code in JavaScript → Get many products**

4. **Create “OpenAI Chat Model”** (LangChain OpenAI chat model node)
   - Model: `gpt-4o-mini`
   - Connect OpenAI credential (API key) under **OpenAi account**

5. **Create “Postgres Chat Memory”** (LangChain Postgres memory node)
   - Session ID type: `Custom Key`
   - Session key expression: `{{ $('Code in JavaScript').item.json.chat_id }}`
   - Context window length: `30`
   - Configure Postgres credential (**Postgres account 2**) to a reachable database.

6. **Create “AI Agent”** (LangChain Agent node)
   - Text input: `{{ $('Telegram Trigger').item.json.message.text }}`
     - (Optional improvement: use `{{ $('Code in JavaScript').item.json.text }}` for normalized input.)
   - Set the **System message** to the SOFÍA instructions, including:
     - Mandatory greeting protocol
     - Catalog precision and focus rules
     - Strict JSON output schema
     - Product list injection expression referencing **Get many products**:
       - `{{ $('Get many products').all().map(p => p.json.title + " (Precio: $" + p.json.variants[0].price + ")").join(', ') }}`
   - Enable output parsing (so the node returns structured output if supported).
   - Connect the special ports:
     - **OpenAI Chat Model (ai_languageModel) → AI Agent**
     - **Postgres Chat Memory (ai_memory) → AI Agent**
   - Connect main flow:
     - **Get many products → AI Agent**

7. **Create “Code in JavaScript 2”** (`Code` node)
   - Implement JSON extraction/parse from agent output (`items[0].json.output`)
   - Add fallback JSON object if parsing fails.
   - **Important:** If your n8n requires item arrays, return:
     - `return [{ json: parsed }];` and `return [{ json: fallback }];`
   - Connect: **AI Agent → Code in JavaScript 2**

8. **Create “Send a text message”** (`Telegram` node)
   - Operation: Send Message
   - Chat ID: `{{ $('Code in JavaScript').first().json.chat_id }}`
   - Text: `{{ $json.response }}`
   - Connect Telegram credential **Telegram account**
   - Connect: **Code in JavaScript 2 → Send a text message**

9. **Test**
   - Activate workflow
   - Send a Telegram text message to the bot
   - Verify:
     - First-time user gets the name question
     - Returning user gets personalized greeting
     - Product listing matches Shopify titles/prices

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow turns a generic chatbot into an AI advisor that remembers users and responds using real business data. It combines user messages, persistent memory, and live product data to generate accurate and human-like responses. | Sticky note: “AI Advisor with Memory and Real-Time Context” |
| How to set up: Import workflow, connect Telegram/OpenAI/Shopify/Postgres credentials, configure DB for chat memory, test via Telegram message. | Sticky note: “AI Advisor with Memory and Real-Time Context” |
| Customization: Adjust system message tone/industry, replace Shopify with another data source, change memory window size. | Sticky note: “AI Advisor with Memory and Real-Time Context” |
| ⚠️ Important: This workflow requires valid JSON output from the AI. If you modify the system prompt, make sure the response format is preserved to avoid parsing errors. | Sticky note: “⚠️ Important” |