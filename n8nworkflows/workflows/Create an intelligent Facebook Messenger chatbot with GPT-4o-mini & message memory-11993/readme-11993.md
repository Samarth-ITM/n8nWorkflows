Create an intelligent Facebook Messenger chatbot with GPT-4o-mini & message memory

https://n8nworkflows.xyz/workflows/create-an-intelligent-facebook-messenger-chatbot-with-gpt-4o-mini---message-memory-11993


# Create an intelligent Facebook Messenger chatbot with GPT-4o-mini & message memory

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow implements a “smart” Facebook Messenger chatbot that receives incoming Messenger webhook events, filters valid user text messages, batches rapid consecutive messages into a single turn, runs an AI Agent powered by **OpenAI GPT-4o-mini** with **per-user conversation memory**, and sends the response back via the Facebook Graph API.

**Target use cases:**
- Customer support automation for businesses on Facebook Pages
- Lead qualification / FAQ handling in Messenger
- AI-assisted conversational interface with short-term memory

**Logical blocks:**
1.1 **Webhook Verification (GET)** — validates Facebook’s webhook verification request and returns the `hub.challenge` if the verify token matches.  
1.2 **Message Reception & Filtering (POST)** — acknowledges events immediately and filters out echo / non-text / empty messages.  
1.3 **Message Batching & User Feedback** — stores messages per user in static data, marks messages as seen, waits 3 seconds, then merges the batch.  
1.4 **AI Processing with Memory** — typing indicator + LangChain AI Agent using GPT-4o-mini and buffer-window memory keyed by user ID.  
1.5 **Response Formatting & Delivery** — removes markdown, truncates for Messenger limits, sends message to user via Graph API.

---

## 2. Block-by-Block Analysis

### 2.1 Webhook Verification (GET)

**Overview:** Handles Facebook’s webhook verification handshake. Facebook sends `hub.verify_token` and expects the workflow to echo `hub.challenge` when the token is correct, otherwise return 403.

**Nodes involved:**
- Facebook Verification Webhook
- Is Token Valid?
- Respond with Challenge
- Respond Forbidden

#### Node: Facebook Verification Webhook
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — entry point for verification requests.
- **Configuration (interpreted):**
  - **Path:** `facebook-messenger-webhook`
  - **HTTP method:** default (GET) since POST is explicitly defined on the other webhook node.
  - **Response mode:** `responseNode` (a Respond to Webhook node must answer).
- **Inputs/Outputs:** No inputs. Output goes to **Is Token Valid?**
- **Version notes:** typeVersion 2.
- **Failure/edge cases:**
  - If Facebook calls a different path than configured, verification fails.
  - If your n8n URL is not publicly reachable (HTTPS required by Facebook), verification fails.

#### Node: Is Token Valid?
- **Type / role:** `IF` — checks `hub.verify_token`.
- **Configuration:**
  - Condition: `{{$json.query['hub.verify_token']}}` **equals** `YOUR_VERIFY_TOKEN_HERE`
  - Case sensitive; strict validation.
- **Connections:**
  - **True** → Respond with Challenge
  - **False** → Respond Forbidden
- **Failure/edge cases:**
  - If the verify token is not updated from the placeholder, verification will always fail.
  - If query params are missing (unexpected request), expression resolves to undefined; condition will fail (go False).

#### Node: Respond with Challenge
- **Type / role:** `Respond to Webhook` — returns the challenge string to Facebook.
- **Configuration:**
  - Respond with: Text
  - Response body: `{{$json.query['hub.challenge']}}`
- **Failure/edge cases:**
  - Missing `hub.challenge` will return empty response → verification fails.

#### Node: Respond Forbidden
- **Type / role:** `Respond to Webhook` — returns 403.
- **Configuration:**
  - HTTP status: 403
  - Body: `Verification failed`
- **Failure/edge cases:** None significant beyond normal webhook response.

---

### 2.2 Message Reception & Filtering (POST)

**Overview:** Receives incoming Messenger events (POST), immediately acknowledges them (required by Facebook), then filters to keep only genuine user text messages (not echoes).

**Nodes involved:**
- Facebook Message Webhook
- Acknowledge Event
- Filter Valid Messages

#### Node: Facebook Message Webhook
- **Type / role:** `Webhook` — entry point for message events.
- **Configuration:**
  - **Path:** `facebook-messenger-webhook` (same as GET webhook; differentiated by method)
  - **HTTP method:** POST
  - **Response mode:** responseNode
- **Connections:** → Acknowledge Event
- **Edge cases:**
  - Messenger can send many event types (delivery, read, postback, attachments). This workflow only processes text messages; others will be filtered out later.

#### Node: Acknowledge Event
- **Type / role:** `Respond to Webhook` — immediately returns 200 to Facebook.
- **Configuration:**
  - Status code: 200
  - Body: `EVENT_RECEIVED`
- **Why it matters:** Facebook expects quick acknowledgment; long processing before responding can cause retries/timeouts.
- **Connections:** → Filter Valid Messages

#### Node: Filter Valid Messages
- **Type / role:** `IF` — filters out unwanted messages.
- **Configuration (conditions, AND):**
  1. “has-message”: checks existence of `{{$json.body?.entry?.[0]?.messaging?.[0]?.message?.text}}`
  2. “not-echo”: `{{$json.body?.entry?.[0]?.messaging?.[0]?.message?.is_echo}}` **notEquals** `true`
- **Connections:**
  - **True** → Store Message for Batching
  - **False** → stops (no further actions)
- **Edge cases / failure types:**
  - If Facebook changes payload shape or sends non-standard events, the optional chaining returns undefined and the message is ignored.
  - Echo handling prevents responding to your own outgoing messages (avoids loops).

---

### 2.3 Message Batching & User Feedback

**Overview:** Stores incoming messages per user in workflow static data, sends “mark_seen”, waits 3 seconds for additional messages, then retrieves and merges all queued messages into a single combined prompt.

**Nodes involved:**
- Store Message for Batching
- Send Seen Indicator
- Wait 3 Seconds
- Retrieve Batched Messages
- Has Messages to Process?

#### Node: Store Message for Batching
- **Type / role:** `Code` — extracts Messenger identifiers and stores message in `global` workflow static data.
- **Key logic/config:**
  - Extracts:
    - `userId` = sender id
    - `pageId` = recipient id (the Page)
    - `messageText`, `timestamp`, `messageId`
  - Uses `$getWorkflowStaticData('global')`
  - Creates/updates `staticData.messageBatches[userId] = { messages: [...], firstMessageTime, pageId }`
  - Pushes `{text, timestamp, messageId}` into `messages`
  - Outputs: `{ userId, pageId, messageText, timestamp, messageId, batchCount }`
- **Connections:** → Send Seen Indicator
- **Edge cases / risks:**
  - **Static data is shared** at workflow level; high concurrency can create race conditions if many messages from same user arrive simultaneously.
  - If `userId` is undefined (unexpected payload), it will index `messageBatches[undefined]`, polluting state.
  - Static data persists across executions; if a later step fails before cleanup, batches may remain until next message triggers cleanup logic.

#### Node: Send Seen Indicator
- **Type / role:** `HTTP Request` — calls Facebook Graph API to mark messages as seen.
- **Configuration:**
  - POST `https://graph.facebook.com/v21.0/me/messages`
  - JSON body includes:
    - recipient id = `{{$json.userId}}`
    - `sender_action`: `mark_seen`
  - Auth: predefined credential type `facebookGraphApi`
  - **onError:** `continueRegularOutput`
  - **alwaysOutputData:** true
- **Connections:** → Wait 3 Seconds
- **Failure types:**
  - Token invalid/expired → 400/401 errors (but flow continues due to continueRegularOutput).
  - Missing `pages_messaging` permissions or wrong token type → API errors.

#### Node: Wait 3 Seconds
- **Type / role:** `Wait` — buffers time to allow additional messages to arrive before processing.
- **Configuration:** amount = 3 seconds
- **Connections:** → Retrieve Batched Messages
- **Edge cases:**
  - If user sends many messages continuously, this design processes after each wait cycle; batching window is fixed at 3 seconds from this execution, not “debounced” across multiple new arrivals.

#### Node: Retrieve Batched Messages
- **Type / role:** `Code` — reads the stored batch and produces a single combined message.
- **Key expressions/variables:**
  - Reads `userId` and `pageId` from: `$('Store Message for Batching').first().json`
    - This is intentional because the immediate input after HTTP request may not contain the original JSON.
  - Looks up `staticData.messageBatches?.[userId]`
  - If empty: returns `{skip: true}`
  - Else:
    - sorts messages by timestamp
    - joins texts with a space: `combinedMessage`
    - deletes the batch entry: `delete staticData.messageBatches[userId]`
  - Outputs: `{ userId, pageId, combinedMessage, messageCount, skip: false }`
- **Connections:** → Has Messages to Process?
- **Edge cases / failure types:**
  - If the Store node didn’t run (unexpected), `$('Store Message for Batching')...` can throw (expression resolution issues).
  - If another execution already deleted the batch (race), it returns `skip: true`.
  - Joining with spaces may lose intentional formatting or sentence boundaries; also merges multiple user intents into one.

#### Node: Has Messages to Process?
- **Type / role:** `IF` — gate to proceed only when `skip` is false.
- **Configuration:** `{{$json.skip}} equals false`
- **Connections:**
  - **True** → Send Typing Indicator
  - **False** → stops
- **Edge cases:** None beyond upstream correctness.

---

### 2.4 AI Processing with Memory

**Overview:** Sends “typing_on”, then invokes an AI Agent that uses GPT-4o-mini and a per-user memory window (last 50 messages) keyed by user ID.

**Nodes involved:**
- Send Typing Indicator
- OpenAI Chat Model
- Conversation Memory
- AI Agent

#### Node: Send Typing Indicator
- **Type / role:** `HTTP Request` — tells Messenger the bot is typing.
- **Configuration:**
  - POST `https://graph.facebook.com/v21.0/me/messages`
  - JSON body: recipient id from `{{$json.userId}}`, `sender_action: typing_on`
  - Auth: `facebookGraphApi`
  - onError: continueRegularOutput; alwaysOutputData: true
- **Connections:** → AI Agent
- **Failure types:** Same as “seen” indicator; failures won’t block processing.

#### Node: OpenAI Chat Model
- **Type / role:** LangChain chat model connector (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Uses OpenAI credentials configured in n8n (API key).
- **Connections:**
  - Provides **ai_languageModel** input to AI Agent.
- **Failure types:**
  - Missing/invalid OpenAI API key; quota exceeded; model not available to the account.
  - Timeouts or rate limits under load.

#### Node: Conversation Memory
- **Type / role:** Memory buffer window (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)
- **Configuration:**
  - Session id type: customKey
  - **sessionKey:** `{{$('Retrieve Batched Messages').first().json.userId}}`
  - contextWindowLength: 50 (last 50 messages)
- **Connections:**
  - Provides **ai_memory** input to AI Agent.
- **Edge cases:**
  - If `userId` is missing, multiple users could share the same memory key (or memory may break).
  - Memory is short-term window; it does not persist “forever context”, only last 50 turns.

#### Node: AI Agent
- **Type / role:** LangChain agent (`@n8n/n8n-nodes-langchain.agent`) — generates the assistant response.
- **Configuration:**
  - Prompt type: define
  - User text input: `{{$('Retrieve Batched Messages').first().json.combinedMessage}}`
  - System message enforces:
    - concise, friendly, professional
    - under 2000 chars
    - Messenger-appropriate style
- **Connections:** main output → Format Response
- **Failure types / edge cases:**
  - Agent output shape may vary; downstream code handles `output` or `text`.
  - Model may still exceed desired length; truncation happens later.
  - If you later add tools/actions to the agent, ensure the output remains compatible with formatting node.

---

### 2.5 Response Formatting & Delivery

**Overview:** Normalizes the AI output for Messenger (removes markdown and code formatting), truncates to a safe length, then sends via Graph API.

**Nodes involved:**
- Format Response
- Send Response to User
- Success

#### Node: Format Response
- **Type / role:** `Code` — prepares final message text.
- **Key logic:**
  - Reads AI response from: `$input.first().json.output || $input.first().json.text || ''`
  - Gets `userId`/`pageId` from Retrieve Batched Messages node
  - Trims and truncates at **1900 chars** (buffer under Messenger 2000 limit)
  - Removes markdown constructs:
    - `**bold**`, `*italic*`, inline code, fenced code blocks
  - Outputs: `{ userId, pageId, response }`
- **Connections:** → Send Response to User
- **Edge cases:**
  - Truncation can cut sentences mid-way.
  - Markdown stripping regex can behave unexpectedly for nested/complex markdown.

#### Node: Send Response to User
- **Type / role:** `HTTP Request` — sends the message back through Facebook Graph API.
- **Configuration:**
  - POST `https://graph.facebook.com/v21.0/me/messages`
  - Body:
    - recipient id = `{{$json.userId}}`
    - messaging_type = `RESPONSE`
    - message.text = `{{ JSON.stringify($json.response) }}` (ensures proper JSON escaping)
  - Auth: `facebookGraphApi`
  - onError: continueRegularOutput; alwaysOutputData: true
- **Connections:** → Success
- **Failure types:**
  - Invalid/expired Page Access Token, permissions missing, or Page not subscribed to webhook events.
  - User may have blocked the Page or cannot be messaged (Graph API error).

#### Node: Success
- **Type / role:** `Set` — terminal node (currently does not set fields).
- **Configuration:** no explicit fields; effectively a “done” marker.
- **Edge cases:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | Sticky Note | Documentation / overview panel | — | — | # Facebook Messenger AI Chatbot … (includes Setup Requirements + Quick Start) |
| Sticky Note - Webhook Verification | Sticky Note | Documentation for verification block | — | — | ## 1. Webhook Verification … ⚠️ Change `YOUR_VERIFY_TOKEN_HERE` … |
| Sticky Note - Message Receipt | Sticky Note | Documentation for receipt/filtering | — | — | ## 2. Receive & Filter Messages … |
| Sticky Note - Message Batching | Sticky Note | Documentation for batching feature | — | — | ## 3. Message Batching (Smart Feature!) … |
| Sticky Note - AI Processing | Sticky Note | Documentation for AI block | — | — | ## 4. AI Agent Processing … |
| Sticky Note - OpenAI Setup | Sticky Note | Credential instructions for OpenAI | — | — | ### OpenAI Credential Setup … link: https://platform.openai.com/api-keys |
| Sticky Note - Response | Sticky Note | Documentation for response formatting/sending | — | — | ## 5. Format & Send Response … |
| Sticky Note - Facebook Setup | Sticky Note | Credential instructions for Facebook token | — | — | ### Facebook Graph API Credential Setup … link: https://developers.facebook.com |
| Facebook Verification Webhook | Webhook | GET endpoint for Facebook webhook verification | — | Is Token Valid? | ## 1. Webhook Verification … |
| Is Token Valid? | IF | Validates `hub.verify_token` | Facebook Verification Webhook | Respond with Challenge; Respond Forbidden | ## 1. Webhook Verification … |
| Respond with Challenge | Respond to Webhook | Returns `hub.challenge` to verify webhook | Is Token Valid? (true) | — | ## 1. Webhook Verification … |
| Respond Forbidden | Respond to Webhook | Returns 403 when token invalid | Is Token Valid? (false) | — | ## 1. Webhook Verification … |
| Facebook Message Webhook | Webhook | POST endpoint for Messenger events | — | Acknowledge Event | ## 2. Receive & Filter Messages … |
| Acknowledge Event | Respond to Webhook | Immediate 200 “EVENT_RECEIVED” ack | Facebook Message Webhook | Filter Valid Messages | ## 2. Receive & Filter Messages … |
| Filter Valid Messages | IF | Filters to user text messages, excludes echo | Acknowledge Event | Store Message for Batching | ## 2. Receive & Filter Messages … |
| Store Message for Batching | Code | Stores per-user messages in static data | Filter Valid Messages | Send Seen Indicator | ## 3. Message Batching (Smart Feature!) … |
| Send Seen Indicator | HTTP Request | `mark_seen` via Graph API | Store Message for Batching | Wait 3 Seconds | ## 3. Message Batching (Smart Feature!) … |
| Wait 3 Seconds | Wait | Batch window | Send Seen Indicator | Retrieve Batched Messages | ## 3. Message Batching (Smart Feature!) … |
| Retrieve Batched Messages | Code | Merges stored messages and clears batch | Wait 3 Seconds | Has Messages to Process? | ## 3. Message Batching (Smart Feature!) … |
| Has Messages to Process? | IF | Continues only when batch exists | Retrieve Batched Messages | Send Typing Indicator | ## 3. Message Batching (Smart Feature!) … |
| Send Typing Indicator | HTTP Request | `typing_on` via Graph API | Has Messages to Process? | AI Agent | ## 4. AI Agent Processing … |
| OpenAI Chat Model | LangChain Chat Model (OpenAI) | Provides GPT-4o-mini model | — | AI Agent (ai_languageModel) | ## 4. AI Agent Processing … + OpenAI setup note applies |
| Conversation Memory | LangChain Memory Buffer Window | Per-user short-term memory (50) | — | AI Agent (ai_memory) | ## 4. AI Agent Processing … |
| AI Agent | LangChain Agent | Generates assistant reply with system prompt | Send Typing Indicator (+ model + memory) | Format Response | ## 4. AI Agent Processing … |
| Format Response | Code | Cleans markdown + truncates | AI Agent | Send Response to User | ## 5. Format & Send Response … |
| Send Response to User | HTTP Request | Sends final message via Graph API | Format Response | Success | ## 5. Format & Send Response … |
| Success | Set | End marker | Send Response to User | — | ## 5. Format & Send Response … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: `Smart Facebook Messenger Chatbot with OpenAI` (or your preferred name)
- Ensure workflow settings execution order is default (this workflow uses `executionOrder: v1`).

2) **Add Webhook for verification (GET)**
- Node: **Webhook** → name it `Facebook Verification Webhook`
- Path: `facebook-messenger-webhook`
- Response Mode: **Using “Respond to Webhook” node** (responseNode)

3) **Add token validation**
- Node: **IF** → `Is Token Valid?`
- Condition (String equals):
  - Left: `{{$json.query['hub.verify_token']}}`
  - Right: `YOUR_VERIFY_TOKEN_HERE` (replace with your secret token)
- Connect: `Facebook Verification Webhook` → `Is Token Valid?`

4) **Add verification responses**
- Node: **Respond to Webhook** → `Respond with Challenge`
  - Respond with: Text
  - Body: `{{$json.query['hub.challenge']}}`
- Node: **Respond to Webhook** → `Respond Forbidden`
  - Response Code: 403
  - Respond with: Text
  - Body: `Verification failed`
- Connect:
  - `Is Token Valid?` True → `Respond with Challenge`
  - `Is Token Valid?` False → `Respond Forbidden`

5) **Add Webhook for incoming messages (POST)**
- Node: **Webhook** → `Facebook Message Webhook`
- Path: `facebook-messenger-webhook` (same path)
- HTTP Method: **POST**
- Response Mode: responseNode

6) **Acknowledge Messenger event immediately**
- Node: **Respond to Webhook** → `Acknowledge Event`
- Response code: 200
- Body: `EVENT_RECEIVED`
- Connect: `Facebook Message Webhook` → `Acknowledge Event`

7) **Filter valid messages**
- Node: **IF** → `Filter Valid Messages`
- Conditions (AND):
  - “Exists” check: `{{$json.body?.entry?.[0]?.messaging?.[0]?.message?.text}}`
  - “is_echo not equals true”: `{{$json.body?.entry?.[0]?.messaging?.[0]?.message?.is_echo}} != true`
- Connect: `Acknowledge Event` → `Filter Valid Messages`

8) **Store messages for batching**
- Node: **Code** → `Store Message for Batching`
- Paste logic equivalent to:
  - Extract `userId`, `pageId`, `messageText`, `timestamp`, `messageId`
  - Store into `$getWorkflowStaticData('global').messageBatches[userId].messages[]`
  - Output `{ userId, pageId, ... }`
- Connect: `Filter Valid Messages` True → `Store Message for Batching`

9) **Create Facebook Graph API credential**
- In n8n credentials:
  - Create **Facebook Graph API** credential (type must match `facebookGraphApi` in HTTP nodes)
  - Paste **Page Access Token** (generated in your Facebook App / Page settings)
- Ensure your app has Messenger configured and your Page is subscribed to webhook events.

10) **Send “seen” indicator**
- Node: **HTTP Request** → `Send Seen Indicator`
- Method: POST
- URL: `https://graph.facebook.com/v21.0/me/messages`
- Authentication: **Predefined credential type** → Facebook Graph API credential
- Body type: JSON
- Body:
  - recipient.id = `{{$json.userId}}`
  - sender_action = `mark_seen`
- Set **On Error**: continue (so failures don’t block)
- Connect: `Store Message for Batching` → `Send Seen Indicator`

11) **Wait for more messages**
- Node: **Wait** → `Wait 3 Seconds`
- Amount: 3 seconds
- Connect: `Send Seen Indicator` → `Wait 3 Seconds`

12) **Retrieve and merge batch**
- Node: **Code** → `Retrieve Batched Messages`
- Implement:
  - `userId = $('Store Message for Batching').first().json.userId`
  - Load `staticData.messageBatches[userId]`
  - If empty: output `{skip:true}`
  - Else: sort + join to `combinedMessage`, delete the batch, output `{skip:false, combinedMessage, userId, pageId, messageCount}`
- Connect: `Wait 3 Seconds` → `Retrieve Batched Messages`

13) **Gate: only proceed if there’s a batch**
- Node: **IF** → `Has Messages to Process?`
- Condition: `{{$json.skip}} equals false`
- Connect: `Retrieve Batched Messages` → `Has Messages to Process?`

14) **Send typing indicator**
- Node: **HTTP Request** → `Send Typing Indicator`
- POST `https://graph.facebook.com/v21.0/me/messages`
- JSON body: recipient.id = `{{$json.userId}}`, sender_action = `typing_on`
- Auth: Facebook Graph API credential
- On Error: continue
- Connect: `Has Messages to Process?` True → `Send Typing Indicator`

15) **Create OpenAI credential**
- In n8n credentials:
  - Create **OpenAI** credential
  - Paste API key from: https://platform.openai.com/api-keys

16) **Add LangChain OpenAI chat model**
- Node: **OpenAI Chat Model** → `OpenAI Chat Model`
- Model: `gpt-4o-mini`
- Select your OpenAI credential

17) **Add conversation memory**
- Node: **Memory Buffer Window** → `Conversation Memory`
- SessionId type: custom key
- sessionKey: `{{$('Retrieve Batched Messages').first().json.userId}}`
- Context window length: 50

18) **Add AI Agent**
- Node: **AI Agent** → `AI Agent`
- Text input: `{{$('Retrieve Batched Messages').first().json.combinedMessage}}`
- System message: copy/adapt the workflow’s system prompt (concise, professional, under 2000 chars)
- Connect:
  - Main: `Send Typing Indicator` → `AI Agent`
  - AI language model connection: `OpenAI Chat Model` → `AI Agent` (ai_languageModel)
  - AI memory connection: `Conversation Memory` → `AI Agent` (ai_memory)

19) **Format the response**
- Node: **Code** → `Format Response`
- Implement:
  - Get AI output from `$input.first().json.output || ...json.text`
  - Trim, truncate to 1900 chars
  - Strip markdown patterns
  - Output `{ userId, pageId, response }`
- Connect: `AI Agent` → `Format Response`

20) **Send response to Messenger**
- Node: **HTTP Request** → `Send Response to User`
- POST `https://graph.facebook.com/v21.0/me/messages`
- Auth: Facebook Graph API credential
- JSON body:
  - recipient.id = `{{$json.userId}}`
  - messaging_type = `RESPONSE`
  - message.text = `{{ JSON.stringify($json.response) }}`
- On Error: continue
- Connect: `Format Response` → `Send Response to User`

21) **Add terminal node**
- Node: **Set** → `Success` (no fields needed)
- Connect: `Send Response to User` → `Success`

22) **Facebook App webhook configuration**
- In Facebook Developer settings (Messenger Webhooks):
  - Callback URL: your n8n production webhook URL for `/webhook/facebook-messenger-webhook` (exact base depends on n8n hosting)
  - Verify token: must match the token set in `Is Token Valid?`
  - Subscribe to at least `messages` events (and any others you want later)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI API key creation | https://platform.openai.com/api-keys |
| Facebook app creation / developer console | https://developers.facebook.com |
| Messenger constraint: 2000 character limit per message; workflow truncates at 1900 | Reflected in “Format Response” logic and AI system message |
| Workflow uses global static data for batching (`messageBatches[userId]`), which persists between executions | Be mindful of concurrency and cleanup if executions fail mid-way |
| Webhook path is shared between GET verification and POST message events; method differentiates behavior | Both webhooks use `facebook-messenger-webhook` path |

