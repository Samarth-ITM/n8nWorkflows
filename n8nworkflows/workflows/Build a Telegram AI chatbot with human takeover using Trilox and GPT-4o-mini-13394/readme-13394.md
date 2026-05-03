Build a Telegram AI chatbot with human takeover using Trilox and GPT-4o-mini

https://n8nworkflows.xyz/workflows/build-a-telegram-ai-chatbot-with-human-takeover-using-trilox-and-gpt-4o-mini-13394


# Build a Telegram AI chatbot with human takeover using Trilox and GPT-4o-mini

## 1. Workflow Overview

**Purpose:**  
This workflow builds a Telegram chatbot that answers with GPT-4o-mini and supports **human takeover** via **Trilox**. When a human agent takes control in Trilox, the bot becomes silent to avoid double replies. Human replies from Trilox are forwarded back to Telegram.

**Target use cases:**
- Customer support chatbots on Telegram where AI handles FAQs but escalates complex/unsafe topics
- Teams who want an inbox-style “take over / hand back” flow without race-condition double responses
- Bots supporting both **text and voice messages** (voice is transcribed)

### 1.1 Logical Blocks (by dependency)
1. **Message Intake & Normalization (Telegram → text)**  
2. **Record in Trilox + Pre-check Handler (who controls the conversation)**  
3. **Bot Handling (AI response + structured output + post-check handler race condition)**  
4. **Awaiting Human (courtesy message + logging)**  
5. **Human Agent Reply Routing (Trilox trigger → channel router → Telegram forward)**  
6. **Meta / Placeholders / Unsupported paths**

---

## 2. Block-by-Block Analysis

### Block 1 — Message Intake & Normalization
**Overview:**  
Receives inbound Telegram messages and normalizes them into a single `text` field. Supports either direct text or voice transcription.

**Nodes Involved:**
- Telegram Trigger
- Message Type Router
- Extract Text
- Download Voice File
- Transcribe Voice Message
- Merge Text Input
- Ignore Unsupported

#### Node: **Telegram Trigger**
- **Type / Role:** `telegramTrigger` — entry point; listens to incoming Telegram updates.
- **Config choices:** Listens to `message` updates.
- **Key variables/expressions:** Output used widely, notably:
  - `{{$json.message.chat.id}}` for chat/session identification
  - `{{$json.message.text}}` for text payload
  - `{{$json.message.voice.file_id}}` for voice payload
- **Connections:** → Message Type Router
- **Failure/edge cases:**
  - Telegram credential issues (invalid token, revoked access)
  - Telegram webhook misconfiguration / unreachable n8n instance
  - Non-message updates are ignored because only `message` is subscribed

#### Node: **Message Type Router**
- **Type / Role:** `switch` — routes based on whether message is text, voice, or other.
- **Config choices:**
  - Output `Text` if `message.text` exists
  - Output `Voice` if `message.voice.mime_type` exists
  - Fallback output named `extra` (wired to ignore)
- **Connections:**
  - `Text` → Extract Text
  - `Voice` → Download Voice File
  - `extra` → Ignore Unsupported
- **Failure/edge cases:**
  - Telegram payload variants (photos, stickers, etc.) go to fallback and are ignored
  - If Telegram changes payload shape, “exists” checks may fail

#### Node: **Extract Text**
- **Type / Role:** `set` — normalizes text messages to `json.text`.
- **Config choices:** Sets `text = {{$json.message.text}}`.
- **Connections:** → Merge Text Input (input 0)
- **Failure/edge cases:** If `message.text` is unexpectedly missing/null, `text` becomes empty and downstream AI may respond poorly.

#### Node: **Download Voice File**
- **Type / Role:** `telegram` (resource: file) — downloads the voice file binary from Telegram.
- **Config choices:** Uses `fileId = {{$json.message.voice.file_id}}`.
- **Connections:** → Transcribe Voice Message
- **Failure/edge cases:**
  - Large files / Telegram API timeouts
  - Missing permissions / invalid file_id
  - Binary handling depends on workflow setting (`binaryMode: separate` is enabled globally)

#### Node: **Transcribe Voice Message**
- **Type / Role:** LangChain OpenAI node (`openAi` resource audio, operation transcribe) — converts voice to text.
- **Config choices:** Uses OpenAI credentials; relies on incoming binary from previous node.
- **Connections:** → Merge Text Input (input 1)
- **Failure/edge cases:**
  - OpenAI auth/quota/model availability issues
  - Unsupported audio encoding
  - Latency; long audio may exceed limits
- **Version requirements:** Node typeVersion `1.8` (ensure your n8n supports this LangChain OpenAI node version).

#### Node: **Merge Text Input**
- **Type / Role:** `merge` — merges the two possible sources (text vs transcription) into one stream.
- **Config choices:** Default merge behavior (as configured) to unify branches.
- **Connections:** → Record Visitor Message
- **Failure/edge cases:**
  - If both branches fire (rare, but possible in malformed payloads), merge behavior may duplicate items
  - If transcription output doesn’t provide the expected field structure, downstream references can break (the AI Agent later references `$('Merge Text Input').item.json.text`)

#### Node: **Ignore Unsupported**
- **Type / Role:** `noOp` — sink for unsupported message types.
- **Connections:** None
- **Failure/edge cases:** None (intentional drop).

---

### Block 2 — Record in Trilox + Pre-check Handler
**Overview:**  
Logs every visitor message to Trilox and checks whether the bot or a human is currently handling the conversation.

**Nodes Involved:**
- Record Visitor Message
- Check Handler (Pre-Bot)
- Handler Status Router

#### Node: **Record Visitor Message**
- **Type / Role:** `n8n-nodes-trilox.trilox` — creates/records an inbound visitor message in Trilox.
- **Config choices:**
  - `channel = telegram`
  - `senderType = visitor`
  - `chatId = {{$('Telegram Trigger').item.json.message.chat.id}}`
  - `message = {{$json.text}}` (from normalized intake)
  - `appId` must be selected (currently empty in template)
- **Connections:** → Check Handler (Pre-Bot)
- **Failure/edge cases:**
  - Missing `appId` (most common setup failure)
  - Trilox credential/API key invalid
  - If `text` is empty, conversation logs may be unhelpful but still created

#### Node: **Check Handler (Pre-Bot)**
- **Type / Role:** `n8n-nodes-trilox.trilox` (operation: `checkHandler`) — queries who currently controls the conversation.
- **Config choices:**
  - `chatId` tied to Telegram chat id
  - `channel = telegram` implied by node’s context and consistent chat mapping
  - `appId` required
- **Output expectations:** Returns `handlerType` (used by Switch):
  - `bot`, `awaiting_human`, `assigned_human`
- **Connections:** → Handler Status Router
- **Failure/edge cases:**
  - If conversation doesn’t exist yet, behavior depends on Trilox API (could create implicitly or error)
  - Network/API failures block routing

#### Node: **Handler Status Router**
- **Type / Role:** `switch` — routes based on `handlerType`.
- **Config choices:** Named outputs:
  - `bot` if `{{$json.handlerType}} == "bot"`
  - `awaiting_human` if equals `"awaiting_human"`
  - `assigned_human` if equals `"assigned_human"`
- **Connections:**
  - `bot` → AI Agent
  - `awaiting_human` → Send Awaiting Message
  - `assigned_human` → Bot Stays Silent
- **Failure/edge cases:**
  - Unexpected handlerType value: no explicit fallback output configured here; items may be dropped (or remain unrouted)
  - Type validation is strict; non-string values can fail comparisons

---

### Block 3 — Bot Handling (AI + Race-condition Prevention + Optional Escalation)
**Overview:**  
If the bot is in control, an AI agent generates a JSON-structured response. Then the workflow checks handler status again (post-AI) to prevent sending a bot reply if a human took over while the AI was generating.

**Nodes Involved:**
- AI Agent
- OpenAI Chat Model
- Conversation Memory
- Structured Output Parser
- Check Handler (Post-Bot)
- Handler Status Router 2
- Send Bot Reply
- Suppress Bot Reply
- Record Bot Response
- Needs Human?
- Escalate to Human
- Done (No Escalation)

#### Node: **OpenAI Chat Model**
- **Type / Role:** LangChain chat model provider (`lmChatOpenAi`) — supplies the LLM to the agent.
- **Config choices:** Model = `gpt-4o-mini`.
- **Connections:** Provides `ai_languageModel` input to AI Agent.
- **Failure/edge cases:**
  - OpenAI credential/quota errors
  - Model name not available in account/region
  - Timeouts during peak latency

#### Node: **Conversation Memory**
- **Type / Role:** LangChain memory (`memoryBufferWindow`) — keeps last N messages for continuity.
- **Config choices:**
  - Session key: `{{$('Telegram Trigger').item.json.message.chat.id}}` (one conversation per Telegram chat)
  - Context window length: `10`
- **Connections:** Provides `ai_memory` input to AI Agent.
- **Failure/edge cases:**
  - If multiple parallel executions occur for same chat, memory can be overwritten/out of order
  - Chat id missing would break session key expression

#### Node: **Structured Output Parser**
- **Type / Role:** LangChain structured parser — forces/validates JSON schema.
- **Config choices:** Example schema:
  - `message` (string)
  - `is_human_required` (boolean)
- **Connections:** Provides `ai_outputParser` input to AI Agent.
- **Failure/edge cases:**
  - If the model outputs non-JSON or mismatched schema, parser can fail and stop execution (critical dependency)

#### Node: **AI Agent**
- **Type / Role:** LangChain agent — generates the response using system prompt, memory, model, and structured output.
- **Config choices:**
  - Input text: `{{$('Merge Text Input').item.json.text}}`
  - System prompt: “TechHub” business context + strict JSON schema requirement and escalation rules
  - Output parser enabled (`hasOutputParser: true`)
- **Connections:** → Check Handler (Post-Bot)
- **Output expectations:** `$('AI Agent').item.json.output.message` and `...output.is_human_required`
- **Failure/edge cases:**
  - Any upstream missing `text` leads to weak responses or parser issues
  - Prompt instructs “Always respond in valid JSON”; violations break parsing and thus routing
  - Long user inputs could increase latency; increases likelihood of race condition (handled by post-check)

#### Node: **Check Handler (Post-Bot)**
- **Type / Role:** Trilox `checkHandler` — re-checks who controls after AI finishes.
- **Why it matters:** Prevents the “double reply” problem if a human takes over while AI is thinking.
- **Connections:** → Handler Status Router 2
- **Failure/edge cases:** Same as pre-check; if it fails, bot may not respond even if it should (safe failure mode, but could reduce responsiveness).

#### Node: **Handler Status Router 2**
- **Type / Role:** `switch` — routes based on current handlerType (post-AI).
- **Config choices:**
  - `bot` → Send Bot Reply
  - `awaiting_human` → Suppress Bot Reply
  - `assigned_human` → Suppress Bot Reply
- **Connections:** As above
- **Failure/edge cases:**
  - Any unexpected handlerType becomes unrouted (no fallback configured)

#### Node: **Send Bot Reply**
- **Type / Role:** `telegram` send message — delivers AI response to Telegram.
- **Config choices:**
  - `chatId = {{$('Telegram Trigger').item.json.message.chat.id}}`
  - `text = {{$('AI Agent').item.json.output.message}}`
  - `appendAttribution = false`
- **Connections:** → Record Bot Response
- **Failure/edge cases:**
  - Telegram rate limits, blocked bot, chat not found
  - If AI output is missing `message`, expression fails

#### Node: **Suppress Bot Reply**
- **Type / Role:** `noOp` — intentional sink when bot must remain silent.
- **When triggered:** Human is awaiting or assigned at post-check.

#### Node: **Record Bot Response**
- **Type / Role:** Trilox node — logs the bot’s outgoing message to Trilox.
- **Config choices:**
  - `message = {{$('AI Agent').item.json.output.message}}`
  - `chatId` from Telegram chat id
  - `channel = telegram`
  - `appId` required
- **Connections:** → Needs Human?
- **Failure/edge cases:**
  - If this fails, you lose logging; escalation decision still happens only if node runs successfully (because it’s upstream of Needs Human?)

#### Node: **Needs Human?**
- **Type / Role:** `if` — decides whether to escalate based on AI structured flag.
- **Config choices:** Condition checks:
  - `{{$('AI Agent').item.json.output.is_human_required}}` is **true**
- **Connections:**
  - **True** → Escalate to Human
  - **False** → Done (No Escalation)
- **Failure/edge cases:**
  - If the structured output parser fails, this node is never reached
  - If output is not boolean, strict validation may fail

#### Node: **Escalate to Human**
- **Type / Role:** Trilox node (operation: `escalate`) — moves conversation into an escalated state.
- **Config choices:** `chatId` from Telegram chat id; `appId` required.
- **Connections:** None
- **Failure/edge cases:**
  - If escalation fails, bot may continue responding (depending on handler logic) and cause confusion
  - Ensure Trilox permissions/API key allow escalation

#### Node: **Done (No Escalation)**
- **Type / Role:** `noOp` — terminal node when AI handled successfully.

---

### Block 4 — Awaiting Human (Courtesy Message + Logging)
**Overview:**  
If the conversation is escalated but no human is assigned yet, the workflow sends a courtesy message to the customer and records it in Trilox.

**Nodes Involved:**
- Send Awaiting Message
- Record Awaiting Message

#### Node: **Send Awaiting Message**
- **Type / Role:** Telegram send message — informs the user that a human will reply shortly.
- **Config choices:**
  - Static text: “Thank you for your patience! … — System Message”
  - `chatId` from Telegram trigger
  - `appendAttribution = false`
- **Connections:** → Record Awaiting Message
- **Failure/edge cases:** Telegram delivery failures; chatId missing.

#### Node: **Record Awaiting Message**
- **Type / Role:** Trilox node — logs the courtesy message.
- **Config choices:**
  - `message = {{$json.result.text}}` (records the actual sent Telegram text as returned by Telegram node)
  - `chatId` from Telegram trigger
  - `channel = telegram`
- **Connections:** None
- **Failure/edge cases:**
  - If Telegram node output shape differs, `result.text` may not exist (expression break)
  - appId missing

---

### Block 5 — Assigned Human (Bot Silence)
**Overview:**  
If a human is assigned, the bot does nothing, ensuring no AI response is sent.

**Nodes Involved:**
- Bot Stays Silent

#### Node: **Bot Stays Silent**
- **Type / Role:** `noOp` — terminal silent path.
- **Failure/edge cases:** None.

---

### Block 6 — Human Agent Reply Routing (Trilox → Telegram)
**Overview:**  
When a human replies in Trilox, a Trilox trigger fires. The workflow routes by channel and forwards the message to Telegram (other channels are placeholders).

**Nodes Involved:**
- Trilox Trigger (Agent Reply)
- Channel Router
- Forward to Telegram
- Other Channels (Placeholder)

#### Node: **Trilox Trigger (Agent Reply)**
- **Type / Role:** `n8n-nodes-trilox.triloxTrigger` — entry point for human replies from Trilox inbox.
- **Config choices:** `appId` must be selected.
- **Expected input fields:** At minimum:
  - `channel` (e.g., `telegram`)
  - `chat_id` (target chat on that channel)
  - `message` (agent’s message text)
- **Connections:** → Channel Router
- **Failure/edge cases:**
  - Webhook configuration issues
  - Missing/incorrect `appId`
  - Payload mismatches (different field names) will break routing/forwarding

#### Node: **Channel Router**
- **Type / Role:** `switch` — routes based on `{{$json.channel}}`.
- **Config choices:** Outputs for:
  - `telegram`, `whatsapp`, `messenger`, `instagram`, `widget`, `API`
- **Connections:**
  - `telegram` → Forward to Telegram
  - all others → Other Channels (Placeholder)
- **Failure/edge cases:** No fallback configured; unknown channels are unrouted/dropped.

#### Node: **Forward to Telegram**
- **Type / Role:** Telegram send message — forwards the human agent’s reply to Telegram user.
- **Config choices:**
  - `text = {{$json.message}}`
  - `chatId = {{$json.chat_id}}`
  - `appendAttribution = false`
- **Connections:** None
- **Failure/edge cases:**
  - `chat_id` must match Telegram chat ID used earlier; mismatches mean messages go nowhere
  - Telegram delivery/rate limiting

#### Node: **Other Channels (Placeholder)**
- **Type / Role:** `noOp` — placeholder for non-Telegram channel implementations.

---

### Block 7 — Notes / Documentation Nodes
**Overview:**  
Sticky notes provide operational guidance, setup steps, and extension points. They do not affect execution.

**Nodes Involved:**
- Sticky Note - Setup Guide
- Sticky Note - Message Intake
- Sticky Note - Record & Check
- Sticky Note - Bot Handler
- Sticky Note - Awaiting Human
- Sticky Note - Human Agent Reply
- Sticky Note - Quick Test
- Sticky Note - Channel Note

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram messages | — | Message Type Router | ## 1️⃣ Message Intake … Delete voice branch if not needed. |
| Message Type Router | switch | Route text vs voice vs other | Telegram Trigger | Extract Text; Download Voice File; Ignore Unsupported | ## 1️⃣ Message Intake … Delete voice branch if not needed. |
| Extract Text | set | Normalize text into `text` field | Message Type Router (Text) | Merge Text Input | ## 1️⃣ Message Intake … Delete voice branch if not needed. |
| Download Voice File | telegram | Download Telegram voice binary | Message Type Router (Voice) | Transcribe Voice Message | ## 1️⃣ Message Intake … Delete voice branch if not needed. |
| Transcribe Voice Message | openAi (audio transcribe) | Speech-to-text transcription | Download Voice File | Merge Text Input | ## 1️⃣ Message Intake … Delete voice branch if not needed. |
| Merge Text Input | merge | Unify text input (typed or transcribed) | Extract Text; Transcribe Voice Message | Record Visitor Message | ## 1️⃣ Message Intake … Delete voice branch if not needed. |
| Ignore Unsupported | noOp | Drop unsupported Telegram message types | Message Type Router (extra) | — | ## 1️⃣ Message Intake … Delete voice branch if not needed. |
| Record Visitor Message | trilox | Log inbound visitor message to Trilox | Merge Text Input | Check Handler (Pre-Bot) | ## 2️⃣ Record & Check Handler … handlerType meanings |
| Check Handler (Pre-Bot) | trilox (checkHandler) | Determine who controls conversation (pre-AI) | Record Visitor Message | Handler Status Router | ## 2️⃣ Record & Check Handler … handlerType meanings |
| Handler Status Router | switch | Route by `handlerType` (pre-AI) | Check Handler (Pre-Bot) | AI Agent; Send Awaiting Message; Bot Stays Silent | ## 2️⃣ Record & Check Handler … handlerType meanings |
| AI Agent | langchain.agent | Generate structured response using LLM + memory | Handler Status Router (bot) | Check Handler (Post-Bot) | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| OpenAI Chat Model | lmChatOpenAi | LLM provider for agent (gpt-4o-mini) | — | AI Agent (ai_languageModel) | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Conversation Memory | memoryBufferWindow | Per-chat memory window | — | AI Agent (ai_memory) | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Structured Output Parser | outputParserStructured | Enforce JSON schema output | — | AI Agent (ai_outputParser) | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Check Handler (Post-Bot) | trilox (checkHandler) | Re-check controller after AI completes | AI Agent | Handler Status Router 2 | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Handler Status Router 2 | switch | Suppress bot reply if human took over | Check Handler (Post-Bot) | Send Bot Reply; Suppress Bot Reply | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Send Bot Reply | telegram | Send AI message to Telegram | Handler Status Router 2 (bot) | Record Bot Response | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Suppress Bot Reply | noOp | Silence path when human controls | Handler Status Router 2 (awaiting_human/assigned_human) | — | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Record Bot Response | trilox | Log bot message to Trilox | Send Bot Reply | Needs Human? | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Needs Human? | if | Escalate decision using `is_human_required` | Record Bot Response | Escalate to Human; Done (No Escalation) | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Escalate to Human | trilox (escalate) | Escalate conversation in Trilox | Needs Human? (true) | — | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Done (No Escalation) | noOp | Terminal (no escalation) | Needs Human? (false) | — | ## 3️⃣ Bot Handler … post-check prevents double replies; escalate if needed |
| Send Awaiting Message | telegram | Courtesy message while waiting for agent | Handler Status Router (awaiting_human) | Record Awaiting Message | ## 4️⃣ Awaiting Human … send + log |
| Record Awaiting Message | trilox | Log courtesy message to Trilox | Send Awaiting Message | — | ## 4️⃣ Awaiting Human … send + log |
| Bot Stays Silent | noOp | Terminal silent path when assigned to human | Handler Status Router (assigned_human) | — | ## 2️⃣ Record & Check Handler … handlerType meanings |
| Trilox Trigger (Agent Reply) | triloxTrigger | Entry point: human replies from Trilox inbox | — | Channel Router | ## 5️⃣ Human Agent Reply … route by channel; placeholders for others |
| Channel Router | switch | Route agent reply to correct channel | Trilox Trigger (Agent Reply) | Forward to Telegram; Other Channels (Placeholder) | ## 5️⃣ Human Agent Reply … route by channel; placeholders for others |
| Forward to Telegram | telegram | Send human agent message to Telegram user | Channel Router (telegram) | — | ## 5️⃣ Human Agent Reply … route by channel; placeholders for others |
| Other Channels (Placeholder) | noOp | Placeholder for WhatsApp/Messenger/etc. | Channel Router (others) | — | 💡 Other channels are not connected… add send nodes as needed. |
| Sticky Note - Setup Guide | stickyNote | Documentation | — | — |  |
| Sticky Note - Message Intake | stickyNote | Documentation | — | — |  |
| Sticky Note - Record & Check | stickyNote | Documentation | — | — |  |
| Sticky Note - Bot Handler | stickyNote | Documentation | — | — |  |
| Sticky Note - Awaiting Human | stickyNote | Documentation | — | — |  |
| Sticky Note - Human Agent Reply | stickyNote | Documentation | — | — |  |
| Sticky Note - Quick Test | stickyNote | Documentation | — | — |  |
| Sticky Note - Channel Note | stickyNote | Documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Prerequisites (self-hosted n8n)**
   1. Ensure you are running **self-hosted n8n** (community nodes required).
   2. Install community node: **Settings → Community Nodes → Install** `n8n-nodes-trilox`.
   3. Create accounts/tokens:
      - Trilox: create account at https://trilox.io → create **Project → App (Inbox) → API Key**
      - Telegram: create bot via https://t.me/botfather → get bot token
      - OpenAI: create API key

2. **Create Credentials in n8n**
   1. **Telegram API** credential: paste bot token.
   2. **Trilox API** credential: paste Trilox API key.
   3. **OpenAI API** credential: paste OpenAI API key.

3. **Build Block 1: Telegram intake**
   1. Add **Telegram Trigger** node:
      - Updates: `message`
      - Select Telegram credentials
   2. Add **Switch** node named **Message Type Router**:
      - Rule 1 (Text): condition “exists” on `{{$json.message.text}}`
      - Rule 2 (Voice): condition “exists” on `{{$json.message.voice.mime_type}}`
      - Fallback output: `extra`
   3. Add **Set** node named **Extract Text**:
      - Add field `text` = `{{$json.message.text}}`
   4. Add **Telegram** node named **Download Voice File**:
      - Resource: `file`
      - Field `fileId` = `{{$json.message.voice.file_id}}`
   5. Add **OpenAI (LangChain)** node named **Transcribe Voice Message**:
      - Resource: `audio`
      - Operation: `transcribe`
      - Select OpenAI credentials
      - Ensure it receives binary from “Download Voice File”
   6. Add **Merge** node named **Merge Text Input**:
      - Connect **Extract Text → Merge** (input 0)
      - Connect **Transcribe Voice Message → Merge** (input 1)
   7. Add **No Operation** node named **Ignore Unsupported** and connect `Message Type Router (fallback/extra) → Ignore Unsupported`.

4. **Build Block 2: Trilox logging + pre-handler check**
   1. Add **Trilox** node named **Record Visitor Message**:
      - Operation: record/send message (default “message” behavior)
      - **App**: select your Trilox App (this populates `appId`)
      - `chatId` = `{{$('Telegram Trigger').item.json.message.chat.id}}`
      - `channel` = `telegram`
      - `senderType` = `visitor`
      - `message` = `{{$json.text}}`
   2. Add **Trilox** node named **Check Handler (Pre-Bot)**:
      - Operation: `checkHandler`
      - App: same Trilox App
      - `chatId` = Telegram chat id expression above
   3. Add **Switch** node named **Handler Status Router**:
      - Route `bot` when `{{$json.handlerType}} == "bot"`
      - Route `awaiting_human` when equals `"awaiting_human"`
      - Route `assigned_human` when equals `"assigned_human"`

5. **Build Block 3: AI agent + post-check (race condition)**
   1. Add **OpenAI Chat Model** node:
      - Model: `gpt-4o-mini`
      - Select OpenAI credentials
   2. Add **Conversation Memory (Buffer Window)** node:
      - Session ID type: custom key
      - Session key = `{{$('Telegram Trigger').item.json.message.chat.id}}`
      - Context window length: `10`
   3. Add **Structured Output Parser** node:
      - Schema example with:
        - `message` (string)
        - `is_human_required` (boolean)
   4. Add **AI Agent** node:
      - Prompt/input text = `{{$('Merge Text Input').item.json.text}}`
      - System prompt: paste your business rules, but **must** enforce JSON output schema
      - Enable/attach:
        - Language model input from **OpenAI Chat Model**
        - Memory input from **Conversation Memory**
        - Output parser input from **Structured Output Parser**
   5. Add **Trilox** node named **Check Handler (Post-Bot)**:
      - Operation: `checkHandler`
      - Same `chatId` and App
   6. Add **Switch** node named **Handler Status Router 2**:
      - If `handlerType == bot` → continue to send reply
      - If `awaiting_human` or `assigned_human` → suppress reply
   7. Add **Telegram** node named **Send Bot Reply**:
      - `chatId` = `{{$('Telegram Trigger').item.json.message.chat.id}}`
      - `text` = `{{$('AI Agent').item.json.output.message}}`
      - `appendAttribution = false`
   8. Add **No Operation** node named **Suppress Bot Reply**.
   9. Add **Trilox** node named **Record Bot Response**:
      - App: same
      - `chatId` as above
      - `channel = telegram`
      - `message = {{$('AI Agent').item.json.output.message}}`
   10. Add **IF** node named **Needs Human?**:
       - Condition: `{{$('AI Agent').item.json.output.is_human_required}}` is `true`
   11. Add **Trilox** node named **Escalate to Human**:
       - Operation: `escalate`
       - Same App + chatId
   12. Add **No Operation** node named **Done (No Escalation)**.

6. **Build Block 4: Awaiting human**
   1. Add **Telegram** node named **Send Awaiting Message**:
      - `chatId` = `{{$('Telegram Trigger').item.json.message.chat.id}}`
      - Text = your courtesy message
      - `appendAttribution = false`
   2. Add **Trilox** node named **Record Awaiting Message**:
      - `message` = use the Telegram node’s returned text (commonly `{{$json.result.text}}`)
      - Same App + chatId + `channel = telegram`

7. **Build Block 5: Assigned human silence**
   1. Add **No Operation** node named **Bot Stays Silent**.

8. **Build Block 6: Human replies from Trilox → Telegram**
   1. Add **Trilox Trigger** node named **Trilox Trigger (Agent Reply)**:
      - Select the same Trilox App
   2. Add **Switch** node named **Channel Router**:
      - Route `telegram` when `{{$json.channel}} == "telegram"`
      - Add placeholder outputs for whatsapp/messenger/instagram/widget/api if desired
   3. Add **Telegram** node named **Forward to Telegram**:
      - `chatId = {{$json.chat_id}}`
      - `text = {{$json.message}}`
      - `appendAttribution = false`
   4. Add **No Operation** node named **Other Channels (Placeholder)** and connect non-telegram outputs to it.

9. **Wire the connections (order)**
   - Telegram Trigger → Message Type Router  
   - Router (Text) → Extract Text → Merge Text Input  
   - Router (Voice) → Download Voice File → Transcribe Voice Message → Merge Text Input  
   - Router (extra) → Ignore Unsupported  
   - Merge Text Input → Record Visitor Message → Check Handler (Pre-Bot) → Handler Status Router  
   - Handler Status Router:
     - bot → AI Agent → Check Handler (Post-Bot) → Handler Status Router 2  
       - bot → Send Bot Reply → Record Bot Response → Needs Human?
         - true → Escalate to Human
         - false → Done (No Escalation)
       - awaiting_human/assigned_human → Suppress Bot Reply
     - awaiting_human → Send Awaiting Message → Record Awaiting Message
     - assigned_human → Bot Stays Silent  
   - Trilox Trigger (Agent Reply) → Channel Router → (telegram) Forward to Telegram; (others) Other Channels

10. **Final checks**
   - In **every Trilox node and trigger**, choose your **App** (inbox) so `appId` is set.
   - Confirm expressions resolve (especially `Merge Text Input` → `text`, and AI output paths).
   - Activate workflow; test with text and voice; test takeover in Trilox.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Trilox account and setup required (Project → App/inbox → API key). | https://trilox.io |
| Create Telegram bot and obtain token. | https://t.me/botfather |
| Community node required: `n8n-nodes-trilox` (self-hosted n8n). | n8n Settings → Community Nodes |
| Discord support link from the workflow notes. | https://discord.gg/g9e6YTqmUs |
| Key design principle: post-AI handler check prevents double replies if a human takes over during AI latency. | Implemented via “Check Handler (Post-Bot)” + “Handler Status Router 2” |
| Other channels are placeholders (WhatsApp/Messenger/Instagram/Widget/API). | Connect additional send nodes to Channel Router outputs |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.