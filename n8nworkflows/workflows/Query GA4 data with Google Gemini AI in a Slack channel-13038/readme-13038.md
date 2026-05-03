Query GA4 data with Google Gemini AI in a Slack channel

https://n8nworkflows.xyz/workflows/query-ga4-data-with-google-gemini-ai-in-a-slack-channel-13038


# Query GA4 data with Google Gemini AI in a Slack channel

Disclaimer (as provided): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.*

## 1. Workflow Overview

**Purpose:**  
This workflow lets Slack users query **Google Analytics 4 (GA4)** data in **natural language** by mentioning the bot in a dedicated Slack channel. A **LangChain AI Agent** (powered by **Google Gemini**) converts the request into a GA4 reporting query, retrieves the data using the **Google Analytics Tool** node, and posts a reply back into the original Slack thread.

**Target use cases:**
- Quick “How many sessions last week?” or “Top pages by views yesterday?” questions in Slack
- Lightweight, conversational analytics access for teams without opening GA4
- Guardrailed analytics where the AI is explicitly forbidden to invent or estimate numbers

### 1.1 Input Reception (Slack)
Receives an `app_mention` event in a specific Slack channel and extracts the user’s message.

### 1.2 Message Sanitization
Removes the Slack mention markup (e.g., `<@U123...>`) so the AI sees only the user’s question.

### 1.3 AI Orchestration (Gemini + Memory + Tools)
An AI Agent uses:
- **Gemini** as the language model
- **Simple Memory** to retain limited conversation context
- **GA4 Tool** to fetch data (the only allowed data source for numbers)

### 1.4 Output Delivery (Slack Reply)
Posts the AI Agent’s final answer back to Slack in the same thread as the original message.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Slack)

**Overview:**  
Listens for Slack `app_mention` events in a specific channel and outputs the event payload for downstream processing.

**Nodes involved:**
- Slack Trigger

#### Node: Slack Trigger
- **Type / role:** `n8n-nodes-base.slackTrigger` — event-based trigger for Slack.
- **Key configuration choices:**
  - **Trigger event:** `app_mention` (workflow fires when the bot/app is mentioned)
  - **Channel:** fixed to Slack channel ID `C0AA6U5F095` (cached name: `all-ga4-n8n-llm`)
- **Key data used later:**
  - `channel` (to reply into the same channel)
  - `ts` (timestamp used for thread replies)
  - `text` (raw message including mention markup)
- **Connections:**
  - **Output →** `Edit Fields`
- **Credentials / requirements:**
  - Requires Slack app + OAuth token configured in n8n (`Slack account`)
  - Slack app must be installed in the workspace and have permission to read events in that channel.
- **Edge cases / failures:**
  - Missing Slack permissions (events not delivered, or payload missing fields)
  - Bot not present in the channel (no events / mention behavior differs depending on Slack settings)
  - Slack event retries if n8n webhook response is slow/unavailable

---

### Block 2 — Message Sanitization

**Overview:**  
Cleans the incoming Slack text so the AI receives only the human question, not the mention token.

**Nodes involved:**
- Edit Fields

#### Node: Edit Fields
- **Type / role:** `n8n-nodes-base.set` — data shaping / field assignment.
- **Key configuration choices:**
  - Assigns a new `text` field derived from the trigger payload.
- **Key expression / variable:**
  - `text = {{ $json.text.replace(/<@.*?>/g, '').trim() }}`
  - Removes any substring matching `<@...>` (Slack user/app mention format) and trims whitespace.
- **Connections:**
  - **Input ←** `Slack Trigger`
  - **Output →** `AI Agent`
- **Edge cases / failures:**
  - If `text` is missing or not a string, the expression can fail (rare for valid Slack mention events, but possible with unexpected payloads).
  - Regex `<@.*?>` is greedy across minimal matches but could remove more than intended if Slack changes formatting; typically safe.

---

### Block 3 — AI Orchestration (Gemini + Memory + GA4 Tool)

**Overview:**  
The AI Agent interprets the user’s question, maps it to valid GA4 API metric/dimension names (camelCase), calls the GA4 reporting tool for exact numbers, and generates a final response with date range transparency and “no hallucination” guardrails.

**Nodes involved:**
- Google Gemini Chat Model
- Simple Memory
- Get a report in Google Analytics
- AI Agent

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — LLM chat model provider for the agent.
- **Key configuration choices:**
  - Model: `models/gemini-2.5-pro`
- **Connections:**
  - **Output (ai_languageModel) →** `AI Agent`
- **Credentials / requirements:**
  - Requires Google Gemini/PaLM API credentials in n8n (`Google Gemini(PaLM) Api account`)
- **Edge cases / failures:**
  - API quota exhaustion / rate limiting
  - Model access not enabled for the project
  - Increased latency/timeouts for large prompts

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term conversation memory buffer.
- **Key configuration choices:**
  - **Session ID type:** custom key
  - **Session key:** `JAN_26_FINAL_ACTUAL`
  - **Context window length:** `10` (keeps last 10 turns/messages in memory)
  - **On error:** `continueRegularOutput` (memory failures won’t break the workflow)
- **Connections:**
  - **Output (ai_memory) →** `AI Agent`
- **Important behavioral note:**  
  The session key is **static**, meaning *all users/threads share the same memory* unless changed. This can cause cross-talk between Slack users.
- **Edge cases / failures:**
  - Memory store issues (depending on n8n’s internal handling/version)
  - Privacy/bleed risk due to shared session key
  - Confusing context if multiple conversations happen in parallel

#### Node: Get a report in Google Analytics
- **Type / role:** `n8n-nodes-base.googleAnalyticsTool` — GA4 reporting tool exposed to the AI Agent.
- **Key configuration choices:**
  - **Property ID:** `521315571`
  - **Date range:** `custom`
  - **Start date:** from AI input `Start`
  - **End date:** from AI input `End`
  - **Return all:** enabled (attempts to retrieve full result set)
  - **Metrics:** one metric named via AI input `metric`
  - **Dimensions:** one dimension named via AI input `dimension`
- **Key expressions / variables:**
  - `startDate = $fromAI('Start', '', 'string')`
  - `endDate = $fromAI('End', '', 'string')`
  - Metric name: `$fromAI("metric", "The GA4 metric to retrieve such as sessions or activeUsers")`
  - Dimension name: `$fromAI("dimension", "The GA4 dimension like date or pagePath")`
- **Connections:**
  - **Output (ai_tool) →** `AI Agent`
- **Credentials / requirements:**
  - Requires GA4 OAuth2 credentials (`Google Analytics account 2`)
  - The OAuth user must have access to GA4 property `521315571`
- **Edge cases / failures:**
  - Invalid metric/dimension names (especially if not camelCase) will cause the GA4 request to fail
  - Empty/invalid dates supplied by the agent (Start/End missing or malformed)
  - Permission errors (property access)
  - Sampling/thresholding/privacy restrictions from GA4 for certain dimensions/metrics
  - Large result sets: “returnAll” can increase runtime and memory usage

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM + memory + tools to produce final answer.
- **Key configuration choices:**
  - **Input text:** `{{ $node["Edit Fields"].json["text"] }}`
  - **Prompt type:** “define” with a strong **system message**
  - **Output parser:** enabled (`hasOutputParser: true`)
- **System message (interpreted behavior highlights):**
  - Enforces role: “Senior GA4 Analyst”
  - **Strict anti-hallucination:** must only report numbers returned by GA4 tool; if none, say “0” / “No data found.”
  - Requires date transparency: always include start/end dates used
  - “Schema lookup rule”: map natural language concepts to correct GA4 API names; use valid camelCase
  - Explicit warning: names like `product_revenue` will fail; must use `grossItemRevenue`, etc.
- **Connections:**
  - **Input (main) ←** `Edit Fields`
  - **Input (ai_languageModel) ←** `Google Gemini Chat Model`
  - **Input (ai_memory) ←** `Simple Memory`
  - **Input (ai_tool) ←** `Get a report in Google Analytics`
  - **Output (main) →** `Send a message`
- **Edge cases / failures:**
  - If the agent does not infer Start/End, the GA4 tool may be called with empty dates (likely error)
  - The prompt demands schema correctness, but no explicit “schema tool” node exists; mapping relies on the model’s knowledge + instructions
  - If the user asks for multiple metrics/dimensions, the tool configuration supports only **one** metric and **one** dimension per call (agent may need to choose one or fail to satisfy the query)
  - Output parser errors could cause missing `output` field depending on agent behavior/version

---

### Block 4 — Output Delivery (Slack Reply)

**Overview:**  
Posts the AI Agent’s final text response back into Slack as a thread reply to the triggering message.

**Nodes involved:**
- Send a message

#### Node: Send a message
- **Type / role:** `n8n-nodes-base.slack` — Slack API call to send a message.
- **Key configuration choices:**
  - Posts into **the same channel** as the trigger event:
    - `channelId = {{ $node["Slack Trigger"].json["channel"] }}`
  - Sends text from the agent:
    - `text = {{ $node["AI Agent"].json["output"] }}`
  - Responds in the same **thread**:
    - `thread_ts = {{ $node["Slack Trigger"].json["ts"] }}`
- **Connections:**
  - **Input ←** `AI Agent`
  - **Output →** none (terminal)
- **Credentials / requirements:**
  - Slack API credentials (`Slack account`)
  - Slack bot must have permission to post in the channel and to reply in threads
- **Edge cases / failures:**
  - If `AI Agent` does not return `json.output`, Slack message text may be empty or node may error
  - Slack rate limits
  - Attempting to post into channels where the bot is not a member

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Slack Trigger | n8n-nodes-base.slackTrigger | Entry point: receive Slack app mentions in a channel | — | Edit Fields | ## How it works  This workflow seamlessly integrates Google Analytics 4 (GA4) with Slack, allowing users to query their website data using natural language inside a dedicated Slack channel. Video walkthrough: https://www.youtube.com/watch?v=oWXDc6uASfA |
| Edit Fields | n8n-nodes-base.set | Clean and normalize Slack message text | Slack Trigger | AI Agent | ## Setup steps  1. **Slack trigger**: Configure Slack API inside n8n to watch out for new messages that are sent in a specific channel. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Interpret query, call GA4 tool, generate truthful response | Edit Fields; Google Gemini Chat Model (ai_languageModel); Simple Memory (ai_memory); Get a report in Google Analytics (ai_tool) | Send a message | ## Setup steps  3. **AI agent system prompt**: Create an AI agent that is not allowed to estimate or lie on data... Also map natural language metrics to GA4 metrics... |
| Get a report in Google Analytics | n8n-nodes-base.googleAnalyticsTool | Tool: fetch GA4 report (metric/dimension + date range) | — (called by AI Agent as tool) | AI Agent (ai_tool) | ## Setup steps  2. **Credentials**: Configure credentials for Slack, GA4, Gemini . |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for the agent | — | AI Agent (ai_languageModel) | ## Setup steps  2. **Credentials**: Configure credentials for Slack, GA4, Gemini . |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversation memory window for agent context | — | AI Agent (ai_memory) |  |
| Send a message | n8n-nodes-base.slack | Reply in Slack thread with AI output | AI Agent | — | ## Setup steps  4. **Slack reply**: Send a short natural text inside Slack as a reply... containing the numbers and dates used while fetching data |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Query GA4 data with AI and natural language in Slack channel”**
   - (Optional) Add tags: `slack`, `google analytics`, `ga4`

2. **Add “Slack Trigger”**
   - Node type: **Slack Trigger**
   - Trigger event: **app_mention**
   - Channel: select the dedicated channel (e.g., `all-ga4-n8n-llm`)
   - Credentials: connect a **Slack API** credential (OAuth)
     - Ensure the Slack app is installed in the workspace
     - Ensure permissions allow receiving mention events and reading messages

3. **Add “Edit Fields” (Set node)**
   - Node type: **Set**
   - Add a string field named `text` with expression:
     - `{{$json.text.replace(/<@.*?>/g, '').trim()}}`
   - Connect: **Slack Trigger → Edit Fields**

4. **Add “Google Gemini Chat Model”**
   - Node type: **Google Gemini Chat Model**
   - Model: `models/gemini-2.5-pro`
   - Credentials: configure **Google Gemini (PaLM) API** credential

5. **Add “Simple Memory”**
   - Node type: **Simple Memory (Buffer Window)**
   - Session ID type: **Custom Key**
   - Session key: `JAN_26_FINAL_ACTUAL`
   - Context window length: `10`
   - On Error: **Continue (regular output)**  
   - Note: for production, consider making session key dynamic (e.g., Slack thread `ts`), otherwise all users share memory.

6. **Add “Get a report in Google Analytics”**
   - Node type: **Google Analytics Tool** (GA4 report tool)
   - Credentials: configure **Google Analytics OAuth2**
   - Property ID: set your GA4 property (example used: `521315571`)
   - Date range: **Custom**
   - Start Date expression: `$fromAI('Start', '', 'string')`
   - End Date expression: `$fromAI('End', '', 'string')`
   - Metrics:
     - Add one metric “name” driven by AI: `$fromAI("metric", "The GA4 metric to retrieve such as sessions or activeUsers")`
   - Dimensions:
     - Add one dimension driven by AI: `$fromAI("dimension", "The GA4 dimension like date or pagePath")`
   - Enable: **Return All**

7. **Add “AI Agent”**
   - Node type: **AI Agent (LangChain)**
   - Text/Input: `{{$node["Edit Fields"].json["text"]}}`
   - System message: paste and adapt the guardrail/system content (role, “no estimation”, schema mapping rules, and include start/end dates requirement).
   - Ensure the agent is configured to use:
     - **Language model input:** connect **Google Gemini Chat Model → AI Agent** (ai_languageModel)
     - **Memory input:** connect **Simple Memory → AI Agent** (ai_memory)
     - **Tool input:** connect **Get a report in Google Analytics → AI Agent** (ai_tool)
   - Connect: **Edit Fields → AI Agent**

8. **Add “Send a message” (Slack)**
   - Node type: **Slack**
   - Operation: **Send message**
   - Channel: **by ID/expression**, using the trigger’s channel:
     - `{{$node["Slack Trigger"].json["channel"]}}`
   - Message text:
     - `{{$node["AI Agent"].json["output"]}}`
   - Thread reply:
     - Set `thread_ts = {{$node["Slack Trigger"].json["ts"]}}`
   - Connect: **AI Agent → Send a message**
   - Credentials: reuse the same **Slack API** credential

9. **Activate the workflow**
   - Mention the bot/app in the configured Slack channel and ask a GA4 question with an implied or explicit date range.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works” + setup outline and video walkthrough | https://www.youtube.com/watch?v=oWXDc6uASfA |
| The agent is instructed to never estimate numbers and to only report values returned by GA4 tool | Implemented via AI Agent system message |
| Memory session key is static (`JAN_26_FINAL_ACTUAL`) which can mix conversations across users | Consider using Slack `channel + thread_ts` as a dynamic session key for isolation |