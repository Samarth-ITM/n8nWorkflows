Build a Facebook Messenger AI chatbot with OpenAI and Pinecone RAG

https://n8nworkflows.xyz/workflows/build-a-facebook-messenger-ai-chatbot-with-openai-and-pinecone-rag-12848


# Build a Facebook Messenger AI chatbot with OpenAI and Pinecone RAG

## 1. Workflow Overview

**Purpose:** This workflow implements a Facebook Messenger chatbot that replies to user messages using an **OpenAI chat model** (GPT-4o-mini) and **Pinecone Assistant** for retrieval-augmented generation (RAG). It also includes **webhook verification** (required by Facebook), **message filtering**, **smart batching** of rapid consecutive messages, **conversation memory**, and **Messenger-compatible response formatting**.

**Target use cases:**
- Customer support automation on Facebook Pages
- Document-grounded Q&A (policies, product docs, internal FAQs) via Pinecone Assistant
- Messenger conversational automation with reduced API usage through batching

### 1.1 Webhook Verification (GET)
Facebook validates the webhook endpoint via a GET request using `hub.verify_token` and expects `hub.challenge` returned when valid.

### 1.2 Receive & Filter Messages (POST)
Receives incoming Messenger events, immediately acknowledges (`EVENT_RECEIVED`), and filters out echo/non-text messages.

### 1.3 Message Batching
Stores messages per user in workflow static data, marks message as “seen”, waits 3 seconds to gather additional messages, then combines them into a single prompt.

### 1.4 AI Processing (OpenAI + Memory + Pinecone RAG Tool)
An n8n LangChain Agent uses:
- OpenAI chat model as the LLM,
- Memory buffer window per user (last 50 messages),
- Pinecone Assistant tool for context retrieval and citations behavior.

### 1.5 Format & Send Response
Removes markdown, truncates to Messenger-safe length, and sends response via the Facebook Graph API.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook Verification (GET)

**Overview:** Handles Facebook’s webhook verification handshake. If the verify token matches, the workflow returns the challenge string; otherwise returns HTTP 403.

**Nodes involved:**
- Facebook Verification Webhook
- Is Token Valid?
- Respond with Challenge
- Respond Forbidden

#### Node: Facebook Verification Webhook
- **Type / role:** Webhook trigger (GET) for Facebook verification.
- **Configuration choices:**
  - Path: `facebook-messenger-webhook`
  - Response mode: handled by a Respond to Webhook node (not “On Received”).
- **Inputs/Outputs:** Entry point → outputs to **Is Token Valid?**
- **Failure/edge cases:**
  - If Facebook hits a different URL than configured in the App, verification fails.
  - If TLS/hostname is not publicly accessible, Facebook cannot reach the webhook.
- **Version notes:** Webhook node v2.

#### Node: Is Token Valid?
- **Type / role:** IF node to validate `hub.verify_token`.
- **Key expression / variables:**
  - Left: `{{ $json.query['hub.verify_token'] }}`
  - Right: `YOUR_VERIFY_TOKEN_HERE` (must be replaced)
- **Routing:**
  - True → Respond with Challenge
  - False → Respond Forbidden
- **Failure/edge cases:**
  - If query params are absent, expression evaluates to `undefined` and condition fails (expected).
  - Token mismatch is the most common setup error.
- **Version notes:** IF node v2.

#### Node: Respond with Challenge
- **Type / role:** Responds to the GET verification request with the challenge text.
- **Key expression:**
  - Body: `{{ $json.query['hub.challenge'] }}`
- **Failure/edge cases:**
  - If `hub.challenge` missing, response will be empty and verification will fail.
- **Version notes:** Respond to Webhook v1.1.

#### Node: Respond Forbidden
- **Type / role:** Responds with 403 when verification fails.
- **Configuration choices:**
  - Response code: 403
  - Body: “Verification failed”
- **Version notes:** Respond to Webhook v1.1.

---

### Block 2 — Receive & Filter Messages (POST)

**Overview:** Receives message events from Facebook (POST). Immediately acknowledges to prevent Facebook retries/timeouts, then filters out invalid messages.

**Nodes involved:**
- Facebook Message Webhook
- Acknowledge Event
- Filter Valid Messages

#### Node: Facebook Message Webhook
- **Type / role:** Webhook trigger for Messenger events (POST).
- **Configuration choices:**
  - Path: `facebook-messenger-webhook` (same as GET; differentiated by HTTP method)
  - HTTP Method: POST
  - Response mode: response handled by Respond to Webhook node
- **Connections:** → Acknowledge Event
- **Failure/edge cases:**
  - Facebook may send multiple event types (delivery, read receipts, postbacks). This workflow mainly expects `message.text`.
  - Large payloads or unexpected structure may cause later expressions to be `undefined`.
- **Version notes:** Webhook node v2.

#### Node: Acknowledge Event
- **Type / role:** Respond to Webhook; returns immediately to Facebook.
- **Configuration choices:**
  - HTTP 200, body `EVENT_RECEIVED`
- **Why it matters:** Facebook expects quick acknowledgement; slow workflows may lead to retries.
- **Connections:** → Filter Valid Messages
- **Version notes:** Respond to Webhook v1.1.

#### Node: Filter Valid Messages
- **Type / role:** IF filter to keep only real user text messages.
- **Conditions (AND):**
  1. Message text exists  
     `{{ $json.body?.entry?.[0]?.messaging?.[0]?.message?.text }}`
  2. Not an echo  
     `{{ $json.body?.entry?.[0]?.messaging?.[0]?.message?.is_echo }}` != `true`
- **Connections:** True path → Store Message for Batching (false path is unconnected → dropped)
- **Failure/edge cases:**
  - Non-text messages (attachments) will be discarded.
  - If Facebook changes payload nesting or multiple messaging events arrive in one request, only `[0]` is used; additional events are ignored.
- **Version notes:** IF node v2.

---

### Block 3 — Message Batching

**Overview:** Collects multiple rapid-fire messages from the same user into a single combined prompt to reduce LLM calls and improve context. Uses workflow **global static data** keyed by `userId`.

**Nodes involved:**
- Store Message for Batching
- Send Seen Indicator
- Wait 3 Seconds
- Retrieve Batched Messages
- Has Messages to Process?
- Send Typing Indicator

#### Node: Store Message for Batching
- **Type / role:** Code node to extract sender/page/message fields and append into a per-user batch store.
- **Key logic:**
  - Reads: `body.entry[0].messaging[0]...`
  - Writes to: `$getWorkflowStaticData('global').messageBatches[userId]`
  - Stores: `text`, `timestamp`, `messageId`, `pageId`
- **Outputs:** `userId`, `pageId`, `messageText`, `timestamp`, `messageId`, `batchCount`
- **Connections:** → Send Seen Indicator
- **Failure/edge cases:**
  - If `userId` is missing, it will create a `messageBatches[undefined]` bucket (bad state). Consider adding a guard.
  - Workflow static data persists across executions; stale batches could accumulate if the flow stops before cleanup.
- **Version notes:** Code node v2.

#### Node: Send Seen Indicator
- **Type / role:** HTTP Request to Facebook Graph API to mark message as seen.
- **Configuration choices:**
  - POST `https://graph.facebook.com/v21.0/me/messages`
  - JSON body uses `recipient.id = {{$json.userId}}`, `sender_action = mark_seen`
  - Authentication: predefined credential type `facebookGraphApi`
  - **onError:** continue regular output (won’t break batching if Facebook call fails)
  - **alwaysOutputData:** true
- **Inputs:** from Store Message for Batching
- **Outputs:** to Wait 3 Seconds (regardless of HTTP outcome)
- **Failure/edge cases:**
  - Invalid/expired Page Access Token → 400/401 from Graph API.
  - Missing messaging permissions or wrong token type.
  - API versioning (v21.0) changes could require adjustments.

#### Node: Wait 3 Seconds
- **Type / role:** Wait/delay to allow user to send additional messages.
- **Configuration:** amount = 3 seconds
- **Connections:** → Retrieve Batched Messages
- **Failure/edge cases:**
  - High-volume bots may accumulate many waiting executions.
  - If user continues sending messages, multiple parallel executions may compete; this workflow partially mitigates by retrieving and clearing the batch once.

#### Node: Retrieve Batched Messages
- **Type / role:** Code node to read batched messages and combine them.
- **Key expressions / choices:**
  - Gets `userId` and `pageId` explicitly from the earlier node:  
    `$('Store Message for Batching').first().json.userId`
  - Combines texts sorted by timestamp, joined with spaces.
  - Clears the batch: `delete staticData.messageBatches[userId]`
  - Returns `{ skip: true }` if batch is empty/missing.
- **Connections:** → Has Messages to Process?
- **Failure/edge cases:**
  - If the referenced node name changes (“Store Message for Batching”), this breaks.
  - If multiple executions for same user overlap, one may delete the batch before another retrieves it, causing `skip: true`.

#### Node: Has Messages to Process?
- **Type / role:** IF node that only continues when `skip` is false.
- **Condition:** `{{ $json.skip }}` equals `false`
- **Connections:** True → Send Typing Indicator
- **Failure/edge cases:** If `skip` missing, strict type validation can fail expectedly; ensure Retrieve node always outputs `skip`.

#### Node: Send Typing Indicator
- **Type / role:** HTTP Request to send `typing_on` action.
- **Configuration choices:** Similar to Seen Indicator, with `sender_action = typing_on`
- **onError:** continue regular output; **alwaysOutputData:** true
- **Connections:** → AI Agent1
- **Failure/edge cases:** Same Graph API auth/permission risks as above.

---

### Block 4 — AI Processing (Agent + OpenAI + Memory + Pinecone RAG)

**Overview:** Creates an AI Agent that takes the combined message, consults Pinecone Assistant via a tool for document context, uses memory for continuity, and produces a response.

**Nodes involved:**
- OpenAI Chat Model
- Conversation Memory
- Get context snippets in Pinecone Assistant
- AI Agent1

#### Node: OpenAI Chat Model
- **Type / role:** LangChain chat model provider (OpenAI).
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Credentials: OpenAI API key
- **Connections:** Provides `ai_languageModel` input to **AI Agent1**
- **Failure/edge cases:**
  - Invalid OpenAI key, quota exceeded, model not available in region/account.
  - Latency/timeouts under heavy load.
- **Version notes:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` v1.2.

#### Node: Conversation Memory
- **Type / role:** Memory buffer window (last N messages) per session key.
- **Configuration choices:**
  - Session key: `{{ $('Retrieve Batched Messages').first().json.userId }}`
  - Window length: 50 messages
  - Session id type: customKey
- **Connections:** Provides `ai_memory` to **AI Agent1**
- **Failure/edge cases:**
  - If Retrieve Batched Messages output is missing, sessionKey expression fails.
  - Memory growth considerations: many users → many sessions.
- **Version notes:** memoryBufferWindow v1.3.

#### Node: Get context snippets in Pinecone Assistant
- **Type / role:** Pinecone Assistant community tool node (LangChain tool) used by the agent for RAG.
- **Configuration choices:**
  - `assistantData` includes:
    - `name`: `YOUR_ASSISTANT_NAME` (must be replaced, e.g., `n8n-assistant`)
    - `host`: `YOUR_PINECONE_HOST` (must be replaced with your Assistant host)
  - Adds `sourceTag` metadata: `n8n:n8n_nodes_pinecone_assistant:quickstart`
  - Credentials: Pinecone Assistant API key
- **Connections:** Provides `ai_tool` to **AI Agent1**
- **Failure/edge cases:**
  - Community node not installed → workflow cannot run.
  - Wrong assistant name/host → tool failures or empty retrieval.
  - Pinecone API key invalid/permissions issues.
- **Version notes:** community node v1.

#### Node: AI Agent1
- **Type / role:** LangChain Agent orchestrating tool usage + LLM response.
- **Configuration choices:**
  - Prompt input text: `{{ $('Retrieve Batched Messages').first().json.combinedMessage }}`
  - System message enforces:
    - Use Pinecone tool first for doc-grounded questions
    - Cite sources in natural language
    - Admit when KB lacks info; do not invent
    - Keep under Messenger limit and chat-appropriate style
    - Allows general chat without tool
- **Connections:** Main output → Format Response  
  Receives:
  - `ai_languageModel` from OpenAI Chat Model
  - `ai_memory` from Conversation Memory
  - `ai_tool` from Pinecone Assistant Tool
- **Failure/edge cases:**
  - If the agent can’t call the tool (tool errors), it may respond without retrieval unless the prompt forces strict refusal; current prompt instructs “ALWAYS use tool” for questions, but enforcement is best-effort.
  - Tool latency can slow responses; Messenger still requires that you already acknowledged the webhook (this workflow does).
- **Version notes:** agent node v3.

---

### Block 5 — Format & Send Response

**Overview:** Converts the agent output into Messenger-friendly plain text, truncates to safe length, and sends it back via Graph API.

**Nodes involved:**
- Format Response
- Send Response to User
- Success

#### Node: Format Response
- **Type / role:** Code node that cleans and truncates LLM output and adds routing identifiers.
- **Key logic:**
  - Reads AI response from `$input.first().json.output` or `.text`
  - Pulls `userId/pageId` from Retrieve Batched Messages
  - Truncates over 1900 chars (buffer under 2000 limit)
  - Strips markdown: bold/italic/inline code/code fences
- **Connections:** → Send Response to User
- **Failure/edge cases:**
  - If the agent returns a different schema, `output/text` may be empty.
  - Regex removal of code fences is simplistic; may produce odd formatting on complex markdown.

#### Node: Send Response to User
- **Type / role:** HTTP Request to Facebook Graph API sending a text message.
- **Configuration choices:**
  - POST `https://graph.facebook.com/v21.0/me/messages`
  - Body includes:
    - `recipient.id = {{$json.userId}}`
    - `messaging_type = RESPONSE`
    - `message.text = {{ JSON.stringify($json.response) }}` (ensures proper JSON escaping)
  - Auth: `facebookGraphApi`
  - onError: continue regular output, alwaysOutputData true
- **Connections:** → Success
- **Failure/edge cases:**
  - 24-hour messaging window restrictions (Facebook policy) can block sends depending on page/app use case.
  - Rate limits and token issues.

#### Node: Success
- **Type / role:** Set node (effectively a terminator/placeholder).
- **Configuration choices:** No fields set.
- **Failure/edge cases:** None meaningful.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | Sticky Note | Workflow description and setup requirements | — | — | # Facebook Messenger AI Chatbot with Pinecone RAG… (includes Quick Start and requirements) |
| Sticky Note - Webhook Verification | Sticky Note | Explains GET verification flow | — | — | ## 1. Webhook Verification… Change `YOUR_VERIFY_TOKEN_HERE` |
| Sticky Note - Message Receipt | Sticky Note | Explains POST receipt + filtering | — | — | ## 2. Receive & Filter Messages… |
| Sticky Note - Message Batching | Sticky Note | Explains batching approach | — | — | ## 3. Message Batching (Smart Feature!)… |
| Sticky Note - AI Processing | Sticky Note | Explains agent + RAG components | — | — | ## 4. AI Agent with Pinecone RAG… |
| Sticky Note - Pinecone Setup | Sticky Note | Pinecone Assistant prerequisites | — | — | ### Pinecone Setup… [pinecone.io](https://www.pinecone.io) and node install package |
| Sticky Note - OpenAI Setup | Sticky Note | OpenAI credential steps | — | — | ### OpenAI Credential Setup… [OpenAI API Keys](https://platform.openai.com/api-keys) |
| Sticky Note - Response | Sticky Note | Explains formatting/sending | — | — | ## 5. Format & Send Response… |
| Sticky Note - Facebook Setup | Sticky Note | Facebook app/token prerequisites | — | — | ### Facebook Graph API Credential Setup… [developers.facebook.com](https://developers.facebook.com) |
| Facebook Verification Webhook | Webhook | GET endpoint for Facebook verification | — | Is Token Valid? | ## 1. Webhook Verification… |
| Is Token Valid? | IF | Validate `hub.verify_token` | Facebook Verification Webhook | Respond with Challenge; Respond Forbidden | ## 1. Webhook Verification… |
| Respond with Challenge | Respond to Webhook | Return `hub.challenge` | Is Token Valid? (true) | — | ## 1. Webhook Verification… |
| Respond Forbidden | Respond to Webhook | Return 403 on invalid token | Is Token Valid? (false) | — | ## 1. Webhook Verification… |
| Facebook Message Webhook | Webhook | POST endpoint for incoming messages | — | Acknowledge Event | ## 2. Receive & Filter Messages… |
| Acknowledge Event | Respond to Webhook | Immediate 200 “EVENT_RECEIVED” | Facebook Message Webhook | Filter Valid Messages | ## 2. Receive & Filter Messages… |
| Filter Valid Messages | IF | Keep user text messages, drop echoes | Acknowledge Event | Store Message for Batching | ## 2. Receive & Filter Messages… |
| Store Message for Batching | Code | Persist messages per user for batching | Filter Valid Messages | Send Seen Indicator | ## 3. Message Batching (Smart Feature!)… |
| Send Seen Indicator | HTTP Request | Send `mark_seen` to Messenger | Store Message for Batching | Wait 3 Seconds | ## 3. Message Batching (Smart Feature!)…; ### Facebook Graph API Credential Setup… |
| Wait 3 Seconds | Wait | Delay to batch rapid messages | Send Seen Indicator | Retrieve Batched Messages | ## 3. Message Batching (Smart Feature!)… |
| Retrieve Batched Messages | Code | Combine + clear per-user batch | Wait 3 Seconds | Has Messages to Process? | ## 3. Message Batching (Smart Feature!)… |
| Has Messages to Process? | IF | Continue only if not skipped | Retrieve Batched Messages | Send Typing Indicator | ## 3. Message Batching (Smart Feature!)… |
| Send Typing Indicator | HTTP Request | Send `typing_on` action | Has Messages to Process? | AI Agent1 | ## 3. Message Batching (Smart Feature!)…; ### Facebook Graph API Credential Setup… |
| OpenAI Chat Model | LangChain Chat Model (OpenAI) | LLM provider for agent | — | AI Agent1 (ai_languageModel) | ### OpenAI Credential Setup…; ## 4. AI Agent with Pinecone RAG… |
| Conversation Memory | LangChain Memory Buffer Window | Per-user memory (50 messages) | — | AI Agent1 (ai_memory) | ## 4. AI Agent with Pinecone RAG… |
| Get context snippets in Pinecone Assistant | Pinecone Assistant Tool (community) | RAG retrieval tool for agent | — | AI Agent1 (ai_tool) | ### Pinecone Setup…; ## 4. AI Agent with Pinecone RAG… |
| AI Agent1 | LangChain Agent | Orchestrates tool + LLM + memory | Send Typing Indicator; OpenAI Chat Model; Conversation Memory; Pinecone Tool | Format Response | ## 4. AI Agent with Pinecone RAG… |
| Format Response | Code | Clean/truncate for Messenger | AI Agent1 | Send Response to User | ## 5. Format & Send Response… |
| Send Response to User | HTTP Request | Send final message via Graph API | Format Response | Success | ## 5. Format & Send Response…; ### Facebook Graph API Credential Setup… |
| Success | Set | End marker | Send Response to User | — | ## 5. Format & Send Response… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **Facebook Messenger AI Chatbot with Pinecone RAG**
   - (Optional) Add sticky notes to mirror the documented sections.

2. **Add Webhook (GET) for verification**
   - Node: **Webhook**
   - Name: *Facebook Verification Webhook*
   - Path: `facebook-messenger-webhook`
   - HTTP Method: GET (default)
   - Response mode: **Using “Respond to Webhook” node**

3. **Add token validation**
   - Node: **IF**
   - Name: *Is Token Valid?*
   - Condition: String equals
     - Left value: `{{$json.query['hub.verify_token']}}`
     - Right value: your secret token (replace `YOUR_VERIFY_TOKEN_HERE`)
   - Connect: Verification Webhook → Is Token Valid?

4. **Add verification responses**
   - Node: **Respond to Webhook**
     - Name: *Respond with Challenge*
     - Respond with: Text
     - Body: `{{$json.query['hub.challenge']}}`
   - Node: **Respond to Webhook**
     - Name: *Respond Forbidden*
     - Response code: 403
     - Body: `Verification failed`
   - Connect:
     - IF (true) → Respond with Challenge
     - IF (false) → Respond Forbidden

5. **Add Webhook (POST) for message events**
   - Node: **Webhook**
   - Name: *Facebook Message Webhook*
   - Path: `facebook-messenger-webhook`
   - HTTP Method: POST
   - Response mode: **Using “Respond to Webhook” node**

6. **Add immediate acknowledgement**
   - Node: **Respond to Webhook**
   - Name: *Acknowledge Event*
   - Response code: 200
   - Body: `EVENT_RECEIVED`
   - Connect: Facebook Message Webhook → Acknowledge Event

7. **Filter valid user text (exclude echoes)**
   - Node: **IF**
   - Name: *Filter Valid Messages*
   - Conditions (AND):
     - String “exists”: `{{$json.body?.entry?.[0]?.messaging?.[0]?.message?.text}}`
     - Boolean “not equals true”: `{{$json.body?.entry?.[0]?.messaging?.[0]?.message?.is_echo}}` != `true`
   - Connect: Acknowledge Event → Filter Valid Messages (true output continues)

8. **Store message in workflow static data (batching)**
   - Node: **Code**
   - Name: *Store Message for Batching*
   - Paste logic equivalent to:
     - Extract sender/page/message fields from the webhook payload
     - Append to `$getWorkflowStaticData('global').messageBatches[userId].messages`
     - Output `userId`, `pageId`, `batchCount`
   - Connect: Filter Valid Messages (true) → Store Message for Batching

9. **Create Facebook Graph API credentials**
   - In n8n Credentials:
     - Create **Facebook Graph API** credential
     - Paste the **Page Access Token**
   - Ensure your Facebook App has Messenger configured and the Page token has needed permissions.

10. **Send “seen” indicator**
    - Node: **HTTP Request**
    - Name: *Send Seen Indicator*
    - Method: POST
    - URL: `https://graph.facebook.com/v21.0/me/messages`
    - Authentication: **Predefined Credential Type → Facebook Graph API**
    - Body (JSON):
      - recipient.id = `{{$json.userId}}`
      - sender_action = `mark_seen`
    - Set: **On Error → Continue**
    - Connect: Store Message for Batching → Send Seen Indicator

11. **Wait to batch**
    - Node: **Wait**
    - Name: *Wait 3 Seconds*
    - Amount: 3 seconds
    - Connect: Send Seen Indicator → Wait 3 Seconds

12. **Retrieve and combine batched messages**
    - Node: **Code**
    - Name: *Retrieve Batched Messages*
    - Implement:
      - Read `userId/pageId` from **Store Message for Batching** node output
      - Combine all stored messages for that user (sorted by timestamp)
      - Delete the batch entry after combining
      - Output: `combinedMessage`, `messageCount`, `skip`
    - Connect: Wait 3 Seconds → Retrieve Batched Messages

13. **Gate on “skip”**
    - Node: **IF**
    - Name: *Has Messages to Process?*
    - Condition: boolean equals
      - Left: `{{$json.skip}}`
      - Right: `false`
    - Connect: Retrieve Batched Messages → Has Messages to Process? (true continues)

14. **Send typing indicator**
    - Node: **HTTP Request**
    - Name: *Send Typing Indicator*
    - POST `https://graph.facebook.com/v21.0/me/messages`
    - JSON body: recipient.id = `{{$json.userId}}`, sender_action=`typing_on`
    - Auth: Facebook Graph API credential
    - On Error: Continue
    - Connect: Has Messages to Process? (true) → Send Typing Indicator

15. **Add OpenAI model credential**
    - Create **OpenAI API** credential in n8n with your API key.

16. **Create the OpenAI chat model node**
    - Node: **OpenAI Chat Model** (LangChain)
    - Name: *OpenAI Chat Model*
    - Model: `gpt-4o-mini`
    - Select OpenAI credentials.

17. **Create conversation memory**
    - Node: **Memory Buffer Window** (LangChain)
    - Name: *Conversation Memory*
    - Session ID type: Custom key
    - Session key expression: `{{$('Retrieve Batched Messages').first().json.userId}}`
    - Context window length: 50

18. **Install Pinecone Assistant community node**
    - Install package on your n8n instance:
      - `@pinecone-database/n8n-nodes-pinecone-assistant`
    - Restart n8n if required.

19. **Create Pinecone credentials + Assistant**
    - In Pinecone:
      - Create an **Assistant** (e.g., `n8n-assistant`)
      - Upload documents
      - Copy the Assistant host and API key
    - In n8n Credentials:
      - Create **Pinecone Assistant API** credential with API key.

20. **Add Pinecone Assistant Tool node**
    - Node: **Pinecone Assistant Tool**
    - Name: *Get context snippets in Pinecone Assistant*
    - Configure assistant:
      - Name: your assistant name
      - Host: your Pinecone Assistant host
    - This node must be connected to the agent as an **AI tool**.

21. **Add the AI Agent**
    - Node: **AI Agent** (LangChain)
    - Name: *AI Agent1*
    - Prompt type: “Define”
    - User text expression: `{{$('Retrieve Batched Messages').first().json.combinedMessage}}`
    - System message: include the provided rules (tool-first behavior, citations, refusal when missing, concise chat style).

22. **Wire the AI components**
    - Connect:
      - Send Typing Indicator (main) → AI Agent1 (main)
      - OpenAI Chat Model → AI Agent1 (ai_languageModel connection)
      - Conversation Memory → AI Agent1 (ai_memory connection)
      - Pinecone Tool → AI Agent1 (ai_tool connection)

23. **Format the response**
    - Node: **Code**
    - Name: *Format Response*
    - Implement:
      - Read agent output (`output` or `text`)
      - Trim, strip markdown, truncate to ~1900 chars
      - Output `userId`, `response`
    - Connect: AI Agent1 → Format Response

24. **Send message back to user**
    - Node: **HTTP Request**
    - Name: *Send Response to User*
    - POST `https://graph.facebook.com/v21.0/me/messages`
    - Auth: Facebook Graph API credential
    - JSON body includes:
      - recipient.id = `{{$json.userId}}`
      - messaging_type = `RESPONSE`
      - message.text = `{{JSON.stringify($json.response)}}`
    - On Error: Continue
    - Connect: Format Response → Send Response to User

25. **Add terminator node**
    - Node: **Set**
    - Name: *Success*
    - Connect: Send Response to User → Success

26. **Configure Facebook Webhook subscription**
    - In Facebook Developers:
      - Set webhook callback URL to your n8n webhook URL (production)
      - Set verify token to the same value used in *Is Token Valid?*
      - Subscribe to `messages` events for the Page
    - Activate the workflow and test by messaging the Page.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create Pinecone account and assistant, upload documents, and install the community node `@pinecone-database/n8n-nodes-pinecone-assistant`. | [https://www.pinecone.io](https://www.pinecone.io) |
| Create OpenAI API key and configure OpenAI credentials in n8n. | [https://platform.openai.com/api-keys](https://platform.openai.com/api-keys) |
| Create Facebook App, enable Messenger product, connect a Page, generate Page Access Token, configure webhook URL. | [https://developers.facebook.com](https://developers.facebook.com) |
| Messenger constraints: respond quickly to webhook (this workflow acknowledges immediately) and keep messages under ~2000 characters (this workflow truncates to 1900). | Facebook Messenger platform behavior (operational constraint) |

**Disclaimer (as provided):** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.