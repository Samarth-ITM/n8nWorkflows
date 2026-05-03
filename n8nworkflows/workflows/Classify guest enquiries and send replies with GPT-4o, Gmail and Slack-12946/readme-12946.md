Classify guest enquiries and send replies with GPT-4o, Gmail and Slack

https://n8nworkflows.xyz/workflows/classify-guest-enquiries-and-send-replies-with-gpt-4o--gmail-and-slack-12946


# Classify guest enquiries and send replies with GPT-4o, Gmail and Slack

## 1. Workflow Overview

**Purpose:** Automate hospitality guest enquiry handling. A POST webhook receives an enquiry, an AI agent (GPT-4o-mini via OpenAI) classifies the intent (booking/pricing/availability/policy) and drafts a short acknowledgment message. The workflow then assigns an internal agent + SLA, optionally emails the guest (if they prefer email), and posts a full summary to Slack. A separate error trigger posts failures to Slack.

**Primary use cases:**
- Automatically triage incoming enquiries from a website/booking form
- Provide fast “we received your message” acknowledgments without confirming price/availability
- Route enquiries to the right internal team and track urgency via SLA minutes
- Centralize operational visibility in Slack

### 1.1 Input Reception
Receives guest enquiry payload through a webhook endpoint.

### 1.2 AI Processing (Classification + Draft Reply)
Uses an n8n LangChain Agent with OpenAI chat model and a structured output parser to return strict JSON: `reply_message`, `intent_detected`, `assigned_team`.

### 1.3 Intent Routing + Assignment + SLA
Extracts the detected intent and routes to one of four assignment nodes, adding `assigned_agent` and `sla_minutes`.

### 1.4 Guest Communication + Internal Notifications
If preferred contact is Email, send Gmail reply. In all cases, post a Slack summary; additionally post the AI reply text to Slack on the “non-email” path.

### 1.5 Error Handling
Any workflow error triggers a Slack alert containing node name, error message, and timestamp.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Accepts guest enquiry submissions via HTTP POST. Downstream nodes reference the webhook payload directly for guest details (name/email/message/etc.).

**Nodes Involved:**
- `Webhook Trigger1` (Webhook)

#### Node: Webhook Trigger1
- **Type / Role:** `n8n-nodes-base.webhook` — workflow entry point receiving external requests.
- **Key configuration:**
  - **Path:** `guest-enquiry`
  - **Method:** `POST`
  - **Response mode:** `responseNode` (expects a response node later, but this workflow does not include one; see edge cases).
- **Inputs / Outputs:**
  - **Output:** Main → `AI Agent: Classify Intent & Generate Reply`
- **Key data used later:** `{{$json.body.guest_name}}`, `guest_email`, `preferred_contact`, `enquiry_type`, `guest_message`, `check_in`, `check_out`, etc.
- **Edge cases / failures:**
  - **No response node present:** With `responseMode=responseNode`, callers may hang or get an error depending on n8n version/config. Consider switching to “On Received” or adding a Respond node.
  - Missing fields in `body` will cause blank values in prompts/Slack/email; strict nodes using expressions may still run but produce low-quality output.
  - Public webhook endpoint can be abused; consider auth, rate limiting, or signature verification.

---

### Block 2 — AI Processing (Classification + Reply Draft)
**Overview:** Formats the enquiry into a prompt, sends it through an AI Agent powered by OpenAI, and forces structured JSON output.

**Nodes Involved:**
- `AI Agent: Classify Intent & Generate Reply` (LangChain Agent)
- `OpenAI Chat Model` (LangChain OpenAI chat model)
- `Structured Output Parser1` (Structured output parser)

#### Node: OpenAI Chat Model
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat LLM backend to the agent.
- **Key configuration:**
  - **Model:** `gpt-4o-mini`
  - Uses OpenAI API credentials (“OpenAi account 2”).
- **Connections:**
  - **Output (ai_languageModel):** to `AI Agent: Classify Intent & Generate Reply`
- **Edge cases / failures:**
  - Invalid/expired API key, quota limits, or model access restrictions.
  - Latency/timeouts on large traffic spikes.
  - Model name availability changes.

#### Node: Structured Output Parser1
- **Type / Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces that the agent returns valid JSON matching the schema.
- **Key configuration:**
  - Schema example expects an **array with one object**:
    ```json
    [{
      "reply_message": "...",
      "intent_detected": "booking|pricing|availability|policy",
      "assigned_team": "sales|frontdesk|support"
    }]
    ```
  - This design implies downstream references like `$json.output[0].reply_message`.
- **Connections:**
  - **Output (ai_outputParser):** to `AI Agent: Classify Intent & Generate Reply`
- **Edge cases / failures:**
  - If the model outputs non-JSON or mismatched structure, parsing fails and triggers the error workflow path.
  - If you change schema shape (array vs object), you must update downstream expressions (`output[0]...`).

#### Node: AI Agent: Classify Intent & Generate Reply
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + model + structured parsing.
- **Key configuration (interpreted):**
  - **Prompt text** includes guest details interpolated from webhook body: name, email, preferred contact, enquiry type, message, check-in/out.
  - **System message** enforces behavioral constraints:
    - polite, concise (4–6 lines)
    - no invented prices/availability; no booking confirmation
    - output strictly JSON with `reply_message`, `intent_detected`, `assigned_team`
  - **Output parser enabled** (`hasOutputParser: true`) and connected to `Structured Output Parser1`.
- **Connections:**
  - **Main output:** to `Extract Detected Category`
  - **AI language model input:** from `OpenAI Chat Model`
  - **AI output parser:** from `Structured Output Parser1`
- **Edge cases / failures:**
  - If webhook fields are missing/empty, the agent can misclassify intent or produce awkward replies.
  - If user content includes prompt injection, system message reduces risk but does not eliminate it; consider additional sanitization and/or a moderation step.

---

### Block 3 — Intent Routing Logic
**Overview:** Pulls `intent_detected` from the AI output and routes the execution to the proper assignment/SLA branch.

**Nodes Involved:**
- `Extract Detected Category` (Set)
- `Route by Intent Category` (Switch)

#### Node: Extract Detected Category
- **Type / Role:** `n8n-nodes-base.set` — normalizes intent into a top-level `category` field.
- **Key configuration:**
  - Sets `category = {{$json.output[0].intent_detected}}`
- **Connections:**
  - **Input:** from `AI Agent: Classify Intent & Generate Reply`
  - **Output:** to `Route by Intent Category`
- **Edge cases / failures:**
  - If `output[0]` is missing (parser changed or agent failed), expression resolves to `undefined` and routing may go to fallback.

#### Node: Route by Intent Category
- **Type / Role:** `n8n-nodes-base.switch` — branch by detected category.
- **Key configuration:**
  - Switch value: `{{$json.category}}`
  - Rules:
    - `booking` → Output 1 → `Assign to Booking Team`
    - `pricing` → Output 2 → `Assign to Pricing Team`
    - `availability` → Output 3 → `Assign to Availability Team`
    - `policy` → Output 4 → `Assign to Policy Team`
  - **Fallback output:** 3 (routes unknown values to the *policy* branch per JSON: `fallbackOutput: 3` corresponds to the 4th output index in this node’s internal mapping)
- **Connections:**
  - Outputs to the four assignment nodes listed above.
- **Edge cases / failures:**
  - Any intent outside the allowed set ends up in fallback (policy). If you add intents, update both AI schema guidance and this switch.

---

### Block 4 — Team Assignment & SLA Configuration
**Overview:** Adds operational metadata (assigned agent email and SLA minutes) based on intent category.

**Nodes Involved:**
- `Assign to Booking Team` (Set)
- `Assign to Pricing Team` (Set)
- `Assign to Availability Team` (Set)
- `Assign to Policy Team` (Set)

#### Node: Assign to Booking Team
- **Type / Role:** `n8n-nodes-base.set` — annotate booking enquiries.
- **Key configuration:** `assigned_agent="user@example.com"`, `sla_minutes=15`
- **Connections:** → `Check Guest Contact Preference`
- **Edge cases:** placeholder email must be replaced; otherwise routing/Slack info is misleading.

#### Node: Assign to Pricing Team
- **Type / Role:** Set — annotate pricing enquiries.
- **Key configuration:** `assigned_agent="user@example.com"`, `sla_minutes=15`
- **Connections:** → `Check Guest Contact Preference`
- **Edge cases:** same as above.

#### Node: Assign to Availability Team
- **Type / Role:** Set — annotate availability enquiries.
- **Key configuration:** `assigned_agent="user@example.com"`, `sla_minutes=30`
- **Connections:** → `Check Guest Contact Preference`

#### Node: Assign to Policy Team
- **Type / Role:** Set — annotate policy enquiries.
- **Key configuration:** `assigned_agent="support@example.com"`, `sla_minutes=60`
- **Connections:** → `Check Guest Contact Preference`

---

### Block 5 — Guest Communication & Internal Alerts
**Overview:** Determines whether to email the guest based on their preference, then posts details to Slack for internal follow-up. Also logs the AI reply to Slack on the non-email branch.

**Nodes Involved:**
- `Check Guest Contact Preference` (IF)
- `Send Email Reply to Guest` (Gmail)
- `Post Enquiry Summary to Slack` (Slack)
- `Log AI Reply to Slack` (Slack)

#### Node: Check Guest Contact Preference
- **Type / Role:** `n8n-nodes-base.if` — conditional branching.
- **Key configuration:**
  - Condition: `{{$('Webhook Trigger1').item.json.body.preferred_contact}}` **equals** `"Email"`
  - Strict type validation enabled.
- **Connections:**
  - **True path (Email):** `Send Email Reply to Guest` and `Post Enquiry Summary to Slack`
  - **False path (Non-email):** `Log AI Reply to Slack` and `Post Enquiry Summary to Slack`
- **Edge cases / failures:**
  - Case sensitivity: only exact `"Email"` matches. Values like `"email"` or `"E-mail"` will go false.
  - If `preferred_contact` missing, it goes false.

#### Node: Send Email Reply to Guest
- **Type / Role:** `n8n-nodes-base.gmail` — sends acknowledgment email.
- **Key configuration:**
  - **To:** `{{$('Webhook Trigger1').item.json.body.guest_email}}`
  - **Subject:** `Re: Your Enquiry`
  - **Body:** uses AI output: `{{$json.output[0].reply_message}}` + signature text.
  - **Format:** text email.
  - **Credentials:** Gmail OAuth2 (“Gmail credentials”).
- **Connections:**
  - Input: from IF true path
  - No downstream nodes.
- **Edge cases / failures:**
  - OAuth token expiry / missing Gmail scopes.
  - Invalid guest email address (bounce).
  - If AI output missing, message becomes blank (or expression error depending on n8n settings).

#### Node: Post Enquiry Summary to Slack
- **Type / Role:** `n8n-nodes-base.slack` — posts operational summary.
- **Key configuration:**
  - Posts to channel ID `C09GNB90TED` (“general-information”).
  - Message includes:
    - Guest name/email/phone (note: label says “Preferred Contact” but expression uses `guest_phone`)
    - Detected intent: `{{$json.intent_detected || $json.category}}` (however `intent_detected` is not set at top level elsewhere; typically `category` will be used)
    - Assigned team: `{{$json.output[0].assigned_team}}`
    - Assigned agent: `{{$json.assigned_agent || "Unassigned"}}`
    - SLA minutes: `{{$json.sla_minutes}}`
    - Guest message quoted
  - **Credentials:** Slack API (“Slack account vivek”).
- **Connections:** from both IF branches; no downstream.
- **Edge cases / failures:**
  - If the item reaching this node no longer contains `output[0]...` (due to later node overwrites), the Slack message loses assigned_team. In this workflow, assignment Set nodes add fields but do not remove `output`, so it should remain.
  - Channel ID must exist and token must have permission to post.

#### Node: Log AI Reply to Slack
- **Type / Role:** Slack — posts only the AI reply text (non-email branch).
- **Key configuration:**
  - Channel ID `C09D7N4LGUV` (“all-vivek-workspace”)
  - Text: `{{$json.output[0].reply_message}}`
- **Connections:** from IF false path; no downstream.
- **Edge cases / failures:**
  - Same AI output presence considerations as email node.

---

### Block 6 — Error Handling
**Overview:** If any node in the workflow errors, an error trigger workflow path posts an alert to Slack with debugging info.

**Nodes Involved:**
- `Error Handler Trigger` (Error Trigger)
- `Slack: Send Error Alert` (Slack)

#### Node: Error Handler Trigger
- **Type / Role:** `n8n-nodes-base.errorTrigger` — secondary entry point fired on workflow errors.
- **Key configuration:** none.
- **Connections:** → `Slack: Send Error Alert`
- **Edge cases:** only triggers for executions where error workflow is enabled/available in the environment (depends on n8n setup and execution mode).

#### Node: Slack: Send Error Alert
- **Type / Role:** Slack — sends an error notification.
- **Key configuration:**
  - Channel: `C09GNB90TED`
  - Message includes:
    - `{{$json.node.name}}`
    - `{{$json.error.message}}`
    - `{{$json.timestamp}}`
  - Credentials: Slack API.
- **Edge cases:** Slack credential permissions; if Slack is down, errors won’t be delivered.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview Sticky | Sticky Note | Documentation / overview | — | — | ## 🏨 Automate Guest Enquiry Classification & Replies with Base44 and GPT-4o … (setup steps 1–6) |
| Section: Trigger & AI | Sticky Note | Documentation for trigger+AI section | — | — | ## 📥 Webhook Trigger & AI Classification … |
| Section: Routing Logic | Sticky Note | Documentation for routing section | — | — | ## 🧠 Intent Detection & Assignment … |
| Section: Assignment | Sticky Note | Documentation for assignment/SLA | — | — | ## 📋 Team Assignment & SLA Configuration … |
| Section: Notifications | Sticky Note | Documentation for comms/Slack | — | — | ## 📬 Guest Communication & Internal Alerts … |
| Credentials & Security | Sticky Note | Credential requirements | — | — | ## 🔐 Required Credentials … Replace all email addresses and channel IDs… |
| Webhook Trigger1 | Webhook | Receives guest enquiry payload | — | AI Agent: Classify Intent & Generate Reply | ## 📥 Webhook Trigger & AI Classification … |
| AI Agent: Classify Intent & Generate Reply | LangChain Agent | Classify intent + draft reply JSON | Webhook Trigger1; OpenAI Chat Model (ai); Structured Output Parser1 (ai) | Extract Detected Category | ## 📥 Webhook Trigger & AI Classification … |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM backend for agent | — | AI Agent: Classify Intent & Generate Reply (ai_languageModel) | ## 📥 Webhook Trigger & AI Classification … |
| Structured Output Parser1 | Structured Output Parser (LangChain) | Enforce JSON schema | — | AI Agent: Classify Intent & Generate Reply (ai_outputParser) | ## 📥 Webhook Trigger & AI Classification … |
| Extract Detected Category | Set | Normalize intent to `category` | AI Agent: Classify Intent & Generate Reply | Route by Intent Category | ## 🧠 Intent Detection & Assignment … |
| Route by Intent Category | Switch | Route by `category` | Extract Detected Category | Assign to Booking Team; Assign to Pricing Team; Assign to Availability Team; Assign to Policy Team | ## 🧠 Intent Detection & Assignment … |
| Assign to Booking Team | Set | Assign agent + SLA for booking | Route by Intent Category | Check Guest Contact Preference | ## 📋 Team Assignment & SLA Configuration … |
| Assign to Pricing Team | Set | Assign agent + SLA for pricing | Route by Intent Category | Check Guest Contact Preference | ## 📋 Team Assignment & SLA Configuration … |
| Assign to Availability Team | Set | Assign agent + SLA for availability | Route by Intent Category | Check Guest Contact Preference | ## 📋 Team Assignment & SLA Configuration … |
| Assign to Policy Team | Set | Assign agent + SLA for policy | Route by Intent Category | Check Guest Contact Preference | ## 📋 Team Assignment & SLA Configuration … |
| Check Guest Contact Preference | IF | Decide whether to email guest | Any Assign node | (True) Send Email Reply to Guest + Post Enquiry Summary to Slack; (False) Log AI Reply to Slack + Post Enquiry Summary to Slack | ## 📬 Guest Communication & Internal Alerts … |
| Send Email Reply to Guest | Gmail | Send acknowledgement to guest | Check Guest Contact Preference (true) | — | ## 📬 Guest Communication & Internal Alerts … |
| Post Enquiry Summary to Slack | Slack | Post enquiry summary for staff | Check Guest Contact Preference (true/false) | — | ## 📬 Guest Communication & Internal Alerts … |
| Log AI Reply to Slack | Slack | Post AI reply text (non-email path) | Check Guest Contact Preference (false) | — | ## 📬 Guest Communication & Internal Alerts … |
| Sticky Note8 | Sticky Note | Documentation for error handling | — | — | ## 🚨 Error Handling … |
| Error Handler Trigger | Error Trigger | Start error-notification path | — | Slack: Send Error Alert | ## 🚨 Error Handling … |
| Slack: Send Error Alert | Slack | Send error alert to Slack | Error Handler Trigger | — | ## 🚨 Error Handling … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it (e.g.) “Automate Guest Enquiry Classification & Replies with Base44 and GPT-4o”.

2. **Add Webhook trigger**
   - Add node: **Webhook**
   - Name: `Webhook Trigger1`
   - Method: **POST**
   - Path: `guest-enquiry`
   - Response mode: ideally **On Received** (or add a Respond node later if you keep `responseNode`).

3. **Add the AI model node**
   - Add node: **OpenAI Chat Model** (LangChain)
   - Name: `OpenAI Chat Model`
   - Model: `gpt-4o-mini`
   - Credentials: **OpenAI API** (configure API key in n8n credentials).

4. **Add Structured Output Parser**
   - Add node: **Structured Output Parser** (LangChain)
   - Name: `Structured Output Parser1`
   - Provide schema example shaped as an array with one object:
     - `reply_message` (string)
     - `intent_detected` (booking/pricing/availability/policy)
     - `assigned_team` (sales/frontdesk/support)

5. **Add AI Agent**
   - Add node: **AI Agent** (LangChain Agent)
   - Name: `AI Agent: Classify Intent & Generate Reply`
   - Prompt: include webhook fields (guest_name, guest_email, preferred_contact, enquiry_type, guest_message, check_in, check_out).
   - System message: include constraints (no invented prices, no booking confirmation, short, friendly, JSON-only output).
   - Enable **Output Parser**.
   - Connect:
     - `Webhook Trigger1` → `AI Agent...` (main)
     - `OpenAI Chat Model` → `AI Agent...` (ai_languageModel)
     - `Structured Output Parser1` → `AI Agent...` (ai_outputParser)

6. **Extract intent into a routing field**
   - Add node: **Set**
   - Name: `Extract Detected Category`
   - Add string field `category` = `{{$json.output[0].intent_detected}}`
   - Connect: `AI Agent...` → `Extract Detected Category`

7. **Route by intent**
   - Add node: **Switch**
   - Name: `Route by Intent Category`
   - Value to evaluate: `{{$json.category}}` (string)
   - Add rules for: `booking`, `pricing`, `availability`, `policy`
   - Set fallback to route to policy (or your preferred default)
   - Connect: `Extract Detected Category` → `Route by Intent Category`

8. **Create assignment nodes (agent + SLA)**
   - For each route output, add a **Set** node:
     - `Assign to Booking Team`: `assigned_agent` email + `sla_minutes=15`
     - `Assign to Pricing Team`: `assigned_agent` email + `sla_minutes=15`
     - `Assign to Availability Team`: `assigned_agent` email + `sla_minutes=30`
     - `Assign to Policy Team`: `assigned_agent="support@..."` + `sla_minutes=60`
   - Connect each Switch output to its corresponding Set node.
   - Replace placeholder emails with real internal owners/aliases.

9. **Add contact preference decision**
   - Add node: **IF**
   - Name: `Check Guest Contact Preference`
   - Condition: `{{$('Webhook Trigger1').item.json.body.preferred_contact}}` equals `"Email"` (string)
   - Connect each assignment Set node → `Check Guest Contact Preference`

10. **Add Gmail email reply (true path)**
    - Add node: **Gmail** → operation “Send”
    - Name: `Send Email Reply to Guest`
    - To: `{{$('Webhook Trigger1').item.json.body.guest_email}}`
    - Subject: `Re: Your Enquiry`
    - Body: `{{$json.output[0].reply_message}}` plus signature
    - Credentials: **Gmail OAuth2** (connect Google account; ensure send permissions/scopes).
    - Connect: IF **true** → `Send Email Reply to Guest`

11. **Add Slack summary (both paths)**
    - Add node: **Slack** → operation “Post message”
    - Name: `Post Enquiry Summary to Slack`
    - Choose channel (store channel ID)
    - Message template: include guest details, `category`, `output[0].assigned_team`, `assigned_agent`, `sla_minutes`, and guest message.
    - Credentials: **Slack API** (OAuth/token with permission to post).
    - Connect: IF **true** → `Post Enquiry Summary to Slack`
    - Connect: IF **false** → `Post Enquiry Summary to Slack`

12. **Add Slack AI reply log (false path)**
    - Add node: **Slack** → “Post message”
    - Name: `Log AI Reply to Slack`
    - Text: `{{$json.output[0].reply_message}}`
    - Channel: your internal channel
    - Connect: IF **false** → `Log AI Reply to Slack`

13. **Add error handling path**
    - Add node: **Error Trigger**
    - Name: `Error Handler Trigger`
    - Add node: **Slack** “Post message”
    - Name: `Slack: Send Error Alert`
    - Message: include `{{$json.node.name}}`, `{{$json.error.message}}`, `{{$json.timestamp}}`
    - Connect: `Error Handler Trigger` → `Slack: Send Error Alert`

14. **Validate and test**
    - Send a sample POST to `/webhook/guest-enquiry` with JSON body containing the expected fields.
    - Verify: AI output parses, switch routes correctly, Slack posts include SLA and assignee, Gmail sends only when preferred_contact is Email.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Replace all email addresses and channel IDs with your own before deploying.” | Credentials & Security sticky note |
| Required credentials: Gmail OAuth2, Slack API, OpenAI API | Credentials & Security sticky note |
| Workflow expects webhook form fields: guest_name, guest_email, preferred_contact, enquiry_type, guest_message, check_in, check_out (and optionally guest_phone) | Used across AI prompt, IF condition, Slack templates |
| Error handling posts node name, error message, timestamp to Slack | Error Handling sticky note |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.