Classify support tickets by confidence and route with OpenRouter AI

https://n8nworkflows.xyz/workflows/classify-support-tickets-by-confidence-and-route-with-openrouter-ai-13852


# Classify support tickets by confidence and route with OpenRouter AI

# 1. Workflow Overview

This workflow receives support tickets through an HTTP webhook, normalizes the incoming payload, uses an AI model via OpenRouter to classify the ticket, validates the model output, then routes the ticket based on confidence level and category. It is designed for semi-automated support triage where high-confidence classifications can be routed automatically, medium-confidence cases are flagged for review, and low-confidence cases are sent to a human queue.

Typical use cases:
- Front-door support intake for SaaS or internal IT helpdesks
- AI-assisted ticket triage before assignment
- Controlled automation where confidence determines autonomy level
- Category-based routing into downstream systems such as Slack, Jira, helpdesk tools, or email

## 1.1 Input Reception and Normalization

The workflow starts with a webhook that accepts POST requests. A Code node then standardizes ticket fields such as ID, subject, body, email, and timestamp so downstream nodes always work with a predictable schema.

## 1.2 AI Classification

The normalized ticket is passed to an AI Agent node, which uses an OpenRouter chat model to classify the ticket into one of four categories and assign an urgency and confidence score. The prompt instructs the model to return strict JSON only.

## 1.3 Output Parsing and Validation

A Code node parses the LLM output, extracts JSON if needed, and validates the category and confidence fields. If parsing fails or values are invalid, it falls back to safe defaults.

## 1.4 Confidence-Based Routing

A Switch node evaluates the confidence score:
- High confidence: routed automatically by category
- Medium confidence: flagged for review
- Low confidence: sent to human handling

## 1.5 Category-Based Routing and Webhook Response

High-confidence tickets are routed to Billing, Technical, Sales, or a default General path. All terminal paths return a JSON response to the webhook caller summarizing the ticket ID, route/action, category, and confidence.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Normalization

### Overview
This block receives the incoming HTTP payload and reshapes it into a consistent internal ticket object. It protects later nodes from missing or malformed input by applying defaults and trimming values.

### Nodes Involved
- Webhook - Incoming Ticket
- Normalize Ticket

### Node Details

#### Webhook - Incoming Ticket
- **Type and technical role:** `n8n-nodes-base.webhook`; entry-point trigger for external HTTP requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `route-ticket`
  - Response mode: `responseNode`, meaning the request stays open until a Respond to Webhook node replies
- **Key expressions or variables used:** None directly in the node configuration.
- **Input and output connections:**
  - Input: none, this is the workflow trigger
  - Output: Normalize Ticket
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Invalid HTTP method from client
  - Caller timeout if downstream processing takes too long
  - Payload structure may differ from expected `body` object
  - If no Respond to Webhook node executes, the webhook call may hang or fail
- **Sub-workflow reference:** None

#### Normalize Ticket
- **Type and technical role:** `n8n-nodes-base.code`; transforms raw webhook payload into a normalized schema.
- **Configuration choices:**
  - Reads `body` from the first input item: `$input.first().json.body`
  - Creates:
    - `ticketId`: incoming `ticketId` or generated fallback `TKT-<timestamp>`
    - `subject`: trimmed subject or empty string
    - `body`: takes `body` or `message`, trims it, truncates to 2000 characters
    - `email`: lowercased and trimmed
    - `timestamp`: current ISO datetime
- **Key expressions or variables used:**
  - `$input.first().json.body`
  - `Date.now()`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input: Webhook - Incoming Ticket
  - Output: AI - Classify
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If the webhook payload is not nested under `body`, `raw` may be undefined and the node will fail when accessing properties
  - If `body` or `message` is not a string-like value, `.trim()` assumptions may become unsafe
  - Truncation to 2000 chars may remove important context for classification
- **Sub-workflow reference:** None

---

## 2.2 AI Classification

### Overview
This block sends the normalized ticket to an AI agent and instructs it to produce a strict JSON classification payload. The language model itself is provided by a separate OpenRouter Chat Model node.

### Nodes Involved
- AI - Classify
- OpenRouter Chat Model

### Node Details

#### AI - Classify
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI agent node that generates a classification result from the ticket text.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - Prompt includes subject and body from the normalized ticket
  - Explicit instruction to return only valid JSON with:
    - `category`: `billing|technical|sales|general`
    - `urgency`: `low|normal|urgent`
    - `confidence`: `0.0 to 1.0`
- **Key expressions or variables used:**
  - `{{ $json.subject }}`
  - `{{ $json.body }}`
- **Input and output connections:**
  - Main input: Normalize Ticket
  - AI language model input: OpenRouter Chat Model
  - Main output: Parse + Validate
- **Version-specific requirements:** Type version `1.7`; requires compatible LangChain/AI features in the installed n8n version.
- **Edge cases or potential failure types:**
  - Model may still return non-JSON, markdown, or explanatory text
  - Prompt-following may vary by model
  - API/network/authentication errors from OpenRouter
  - Latency can delay webhook response
  - The model may produce invalid category names or confidence types
- **Sub-workflow reference:** None

#### OpenRouter Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`; provides the LLM backend for the AI agent.
- **Configuration choices:**
  - Model: `google/gemini-3-flash-preview`
  - Uses OpenRouter API credentials
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output via `ai_languageModel` connection to AI - Classify
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Invalid or expired OpenRouter credentials
  - Model deprecation or renaming on OpenRouter
  - Rate limits or quota exhaustion
  - Provider-side moderation or transient inference failures
- **Sub-workflow reference:** None

---

## 2.3 Output Parsing and Validation

### Overview
This block converts the AI output into structured JSON and enforces acceptable values. It also restores original ticket identifiers and email from the normalization step.

### Nodes Involved
- Parse + Validate

### Node Details

#### Parse + Validate
- **Type and technical role:** `n8n-nodes-base.code`; parses model output, validates fields, and applies safe defaults.
- **Configuration choices:**
  - Reads first incoming item
  - If `raw.output` is a string, uses it directly; otherwise stringifies it
  - Uses regex to extract the first JSON object-like substring
  - Tries to parse JSON
  - On failure, falls back to:
    - `category: general`
    - `urgency: normal`
    - `confidence: 0`
  - Validates `category` against:
    - `billing`
    - `technical`
    - `sales`
    - `general`
  - Ensures `confidence` is numeric; otherwise sets `0`
  - Re-attaches:
    - `ticketId` from Normalize Ticket
    - `email` from Normalize Ticket
- **Key expressions or variables used:**
  - `$input.first().json`
  - `raw.output`
  - `text.match(/\{[\s\S]*\}/)`
  - `$('Normalize Ticket').first().json.ticketId`
  - `$('Normalize Ticket').first().json.email`
- **Input and output connections:**
  - Input: AI - Classify
  - Output: Confidence Threshold
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Regex-based JSON extraction may mis-handle nested braces or multiple JSON blocks
  - If the AI node output structure changes and no `output` field exists, parsing behavior may become unpredictable
  - `urgency` is not validated, so invalid urgency values can pass through
  - Direct node reference to `Normalize Ticket` assumes that node always executed in the same run
- **Sub-workflow reference:** None

---

## 2.4 Confidence-Based Routing

### Overview
This block decides how much automation is safe based on the AI confidence score. High-confidence cases continue to automatic category routing, while medium and low-confidence cases are diverted to review or human handling.

### Nodes Involved
- Confidence Threshold
- Flag for Review
- Send to Human Queue

### Node Details

#### Confidence Threshold
- **Type and technical role:** `n8n-nodes-base.switch`; multi-branch conditional routing based on `confidence`.
- **Configuration choices:**
  - Output 1: `High Confidence` when `confidence >= 0.85`
  - Output 2: `Medium Confidence` when `confidence >= 0.6`
  - Output 3: `Low Confidence` when `confidence < 0.6`
  - Output labels renamed for readability
- **Key expressions or variables used:**
  - `={{ $json.confidence }}`
- **Input and output connections:**
  - Input: Parse + Validate
  - Output 1: Route by Category
  - Output 2: Flag for Review
  - Output 3: Send to Human Queue
- **Version-specific requirements:** Type version `3.2`
- **Edge cases or potential failure types:**
  - Because conditions are evaluated in output order, values `>= 0.85` correctly go to High before matching Medium
  - Non-numeric confidence may fail strict type validation; upstream node mitigates this by coercing invalid values to `0`
  - Confidence outside `0..1` is not explicitly clamped
- **Sub-workflow reference:** None

#### Flag for Review
- **Type and technical role:** `n8n-nodes-base.code`; marks medium-confidence tickets for review.
- **Configuration choices:**
  - Adds `action: 'flagged_for_review'`
- **Key expressions or variables used:**
  - `$input.first().json`
- **Input and output connections:**
  - Input: Confidence Threshold
  - Output: Respond
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Minimal logic; failures are unlikely unless input item is missing
- **Sub-workflow reference:** None

#### Send to Human Queue
- **Type and technical role:** `n8n-nodes-base.code`; marks low-confidence tickets for human handling.
- **Configuration choices:**
  - Adds `action: 'sent_to_human'`
- **Key expressions or variables used:**
  - `$input.first().json`
- **Input and output connections:**
  - Input: Confidence Threshold
  - Output: Respond
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Minimal logic; failures are unlikely unless input item is missing
- **Sub-workflow reference:** None

---

## 2.5 Category-Based Routing and Final Response

### Overview
This block routes high-confidence tickets by category to simulated handler nodes, then returns a unified JSON response to the original webhook caller. Any unmatched category falls through to the default General path conceptually, though this is not fully wired in the current JSON.

### Nodes Involved
- Route by Category
- Billing Team
- Technical Team
- Sales Team
- General Inbox
- Respond

### Node Details

#### Route by Category
- **Type and technical role:** `n8n-nodes-base.switch`; splits high-confidence tickets by category.
- **Configuration choices:**
  - Output 1: `Billing` when `category == billing`
  - Output 2: `Technical` when `category == technical`
  - Output 3: `Sales` when `category == sales`
  - Output labels renamed
  - No explicit rule for `general`
- **Key expressions or variables used:**
  - `={{ $json.category }}`
- **Input and output connections:**
  - Input: Confidence Threshold
  - Output 1: Billing Team
  - Output 2: Technical Team
  - Output 3: Sales Team
- **Version-specific requirements:** Type version `3.2`
- **Edge cases or potential failure types:**
  - `general` category is validated upstream but has no connected branch here
  - As configured, general-category high-confidence tickets will not reach General Inbox
  - If no switch rule matches and no fallback output is configured, the item stops and the webhook may never receive a response
- **Sub-workflow reference:** None

#### Billing Team
- **Type and technical role:** `n8n-nodes-base.code`; placeholder for billing-team routing.
- **Configuration choices:**
  - Adds `routed: 'billing_team'`
- **Key expressions or variables used:**
  - `$input.first().json`
- **Input and output connections:**
  - Input: Route by Category
  - Output: Respond
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Placeholder only; no real delivery/integration occurs
- **Sub-workflow reference:** None

#### Technical Team
- **Type and technical role:** `n8n-nodes-base.code`; placeholder for technical-team routing.
- **Configuration choices:**
  - Adds `routed: 'technical_team'`
- **Key expressions or variables used:**
  - `$input.first().json`
- **Input and output connections:**
  - Input: Route by Category
  - Output: Respond
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Placeholder only; no real delivery/integration occurs
- **Sub-workflow reference:** None

#### Sales Team
- **Type and technical role:** `n8n-nodes-base.code`; placeholder for sales-team routing.
- **Configuration choices:**
  - Adds `routed: 'sales_team'`
- **Key expressions or variables used:**
  - `$input.first().json`
- **Input and output connections:**
  - Input: Route by Category
  - Output: Respond
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Placeholder only; no real delivery/integration occurs
- **Sub-workflow reference:** None

#### General Inbox
- **Type and technical role:** `n8n-nodes-base.code`; placeholder for a default/general destination.
- **Configuration choices:**
  - Adds `routed: 'general_inbox'`
- **Key expressions or variables used:**
  - `$input.first().json`
- **Input and output connections:**
  - Output: Respond
  - No incoming connection in the current workflow
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Orphan node in current configuration
  - Intended default route is not actually active
- **Sub-workflow reference:** None

#### Respond
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; sends final JSON back to the HTTP caller.
- **Configuration choices:**
  - Response format: JSON
  - Response body includes:
    - `ticketId`
    - `routed`: uses `routed` or falls back to `action`
    - `category`
    - `confidence`
- **Key expressions or variables used:**
  - `{{ JSON.stringify({ ticketId: $json.ticketId, routed: $json.routed || $json.action, category: $json.category, confidence: $json.confidence }) }}`
- **Input and output connections:**
  - Inputs: Billing Team, Technical Team, Sales Team, Flag for Review, Send to Human Queue, General Inbox
  - Output: none
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**
  - If a branch never reaches Respond, the webhook request may fail or time out
  - If both `routed` and `action` are absent, the `routed` field in the response will be undefined
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Incoming Ticket | Webhook | Receives incoming POST support tickets |  | Normalize Ticket | ## Classification-then-Routing  \n### How it works  \n1. **Webhook** receives an incoming support ticket.  \n2. **Normalize Ticket** (Code node) cleans and standardizes the input.  \n3. **AI - Classify** (AI Agent + OpenRouter) classifies the ticket by category and assigns a confidence score.  \n4. **Parse + Validate** (Code node) checks that the AI output has valid categories and scores.  \n5. **Confidence Threshold** (Switch node) routes by confidence level: high (>0.85) processes autonomously, medium (0.6-0.85) gets flagged for review, and low (<0.6) goes to a human queue.  \n6. **Route by Category** (Switch node) fans high-confidence tickets to Billing, Technical, Sales, or General team handlers.  \n### Setup  \n- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node  \n- Copy the Webhook test URL and send test tickets with varying complexity to see different confidence routing  \n- Adjust confidence thresholds in the **Confidence Threshold** Switch node to match your risk tolerance  \n### Customization  \n- Replace the team handler Code nodes with real integrations (Slack, Jira, email)  \n- Adjust thresholds over time as you validate AI accuracy: lower them to automate more  \n## Receive & Normalize |
| Normalize Ticket | Code | Standardizes ticket fields and applies defaults | Webhook - Incoming Ticket | AI - Classify | ## Classification-then-Routing  \n### How it works  \n1. **Webhook** receives an incoming support ticket.  \n2. **Normalize Ticket** (Code node) cleans and standardizes the input.  \n3. **AI - Classify** (AI Agent + OpenRouter) classifies the ticket by category and assigns a confidence score.  \n4. **Parse + Validate** (Code node) checks that the AI output has valid categories and scores.  \n5. **Confidence Threshold** (Switch node) routes by confidence level: high (>0.85) processes autonomously, medium (0.6-0.85) gets flagged for review, and low (<0.6) goes to a human queue.  \n6. **Route by Category** (Switch node) fans high-confidence tickets to Billing, Technical, Sales, or General team handlers.  \n### Setup  \n- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node  \n- Copy the Webhook test URL and send test tickets with varying complexity to see different confidence routing  \n- Adjust confidence thresholds in the **Confidence Threshold** Switch node to match your risk tolerance  \n### Customization  \n- Replace the team handler Code nodes with real integrations (Slack, Jira, email)  \n- Adjust thresholds over time as you validate AI accuracy: lower them to automate more  \n## Receive & Normalize |
| AI - Classify | LangChain Agent | Uses AI to classify ticket category, urgency, and confidence | Normalize Ticket; OpenRouter Chat Model | Parse + Validate | ## Classification-then-Routing  \n### How it works  \n1. **Webhook** receives an incoming support ticket.  \n2. **Normalize Ticket** (Code node) cleans and standardizes the input.  \n3. **AI - Classify** (AI Agent + OpenRouter) classifies the ticket by category and assigns a confidence score.  \n4. **Parse + Validate** (Code node) checks that the AI output has valid categories and scores.  \n5. **Confidence Threshold** (Switch node) routes by confidence level: high (>0.85) processes autonomously, medium (0.6-0.85) gets flagged for review, and low (<0.6) goes to a human queue.  \n6. **Route by Category** (Switch node) fans high-confidence tickets to Billing, Technical, Sales, or General team handlers.  \n### Setup  \n- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node  \n- Copy the Webhook test URL and send test tickets with varying complexity to see different confidence routing  \n- Adjust confidence thresholds in the **Confidence Threshold** Switch node to match your risk tolerance  \n### Customization  \n- Replace the team handler Code nodes with real integrations (Slack, Jira, email)  \n- Adjust thresholds over time as you validate AI accuracy: lower them to automate more  \n## AI Classification |
| OpenRouter Chat Model | OpenRouter Chat Model | Supplies the LLM used by the AI agent |  | AI - Classify | ## AI Classification |
| Parse + Validate | Code | Extracts and validates AI JSON output | AI - Classify | Confidence Threshold | ## Classification-then-Routing  \n### How it works  \n1. **Webhook** receives an incoming support ticket.  \n2. **Normalize Ticket** (Code node) cleans and standardizes the input.  \n3. **AI - Classify** (AI Agent + OpenRouter) classifies the ticket by category and assigns a confidence score.  \n4. **Parse + Validate** (Code node) checks that the AI output has valid categories and scores.  \n5. **Confidence Threshold** (Switch node) routes by confidence level: high (>0.85) processes autonomously, medium (0.6-0.85) gets flagged for review, and low (<0.6) goes to a human queue.  \n6. **Route by Category** (Switch node) fans high-confidence tickets to Billing, Technical, Sales, or General team handlers.  \n### Setup  \n- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node  \n- Copy the Webhook test URL and send test tickets with varying complexity to see different confidence routing  \n- Adjust confidence thresholds in the **Confidence Threshold** Switch node to match your risk tolerance  \n### Customization  \n- Replace the team handler Code nodes with real integrations (Slack, Jira, email)  \n- Adjust thresholds over time as you validate AI accuracy: lower them to automate more  \n## Confidence Routing |
| Confidence Threshold | Switch | Routes tickets by confidence level | Parse + Validate | Route by Category; Flag for Review; Send to Human Queue | ## Classification-then-Routing  \n### How it works  \n1. **Webhook** receives an incoming support ticket.  \n2. **Normalize Ticket** (Code node) cleans and standardizes the input.  \n3. **AI - Classify** (AI Agent + OpenRouter) classifies the ticket by category and assigns a confidence score.  \n4. **Parse + Validate** (Code node) checks that the AI output has valid categories and scores.  \n5. **Confidence Threshold** (Switch node) routes by confidence level: high (>0.85) processes autonomously, medium (0.6-0.85) gets flagged for review, and low (<0.6) goes to a human queue.  \n6. **Route by Category** (Switch node) fans high-confidence tickets to Billing, Technical, Sales, or General team handlers.  \n### Setup  \n- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node  \n- Copy the Webhook test URL and send test tickets with varying complexity to see different confidence routing  \n- Adjust confidence thresholds in the **Confidence Threshold** Switch node to match your risk tolerance  \n### Customization  \n- Replace the team handler Code nodes with real integrations (Slack, Jira, email)  \n- Adjust thresholds over time as you validate AI accuracy: lower them to automate more  \n## Confidence Routing |
| Route by Category | Switch | Routes high-confidence tickets to team handlers | Confidence Threshold | Billing Team; Technical Team; Sales Team | ## Classification-then-Routing  \n### How it works  \n1. **Webhook** receives an incoming support ticket.  \n2. **Normalize Ticket** (Code node) cleans and standardizes the input.  \n3. **AI - Classify** (AI Agent + OpenRouter) classifies the ticket by category and assigns a confidence score.  \n4. **Parse + Validate** (Code node) checks that the AI output has valid categories and scores.  \n5. **Confidence Threshold** (Switch node) routes by confidence level: high (>0.85) processes autonomously, medium (0.6-0.85) gets flagged for review, and low (<0.6) goes to a human queue.  \n6. **Route by Category** (Switch node) fans high-confidence tickets to Billing, Technical, Sales, or General team handlers.  \n### Setup  \n- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node  \n- Copy the Webhook test URL and send test tickets with varying complexity to see different confidence routing  \n- Adjust confidence thresholds in the **Confidence Threshold** Switch node to match your risk tolerance  \n### Customization  \n- Replace the team handler Code nodes with real integrations (Slack, Jira, email)  \n- Adjust thresholds over time as you validate AI accuracy: lower them to automate more  \n## Category Routing |
| Flag for Review | Code | Marks medium-confidence tickets for review | Confidence Threshold | Respond | ## Category Routing |
| Send to Human Queue | Code | Marks low-confidence tickets for human processing | Confidence Threshold | Respond | ## Category Routing |
| Billing Team | Code | Placeholder route for billing tickets | Route by Category | Respond | ## Category Routing |
| Technical Team | Code | Placeholder route for technical tickets | Route by Category | Respond | ## Category Routing |
| Sales Team | Code | Placeholder route for sales tickets | Route by Category | Respond | ## Category Routing |
| General Inbox | Code | Placeholder general destination |  | Respond | ## Category Routing |
| Respond | Respond to Webhook | Returns final JSON response to the webhook caller | Billing Team; Technical Team; Sales Team; Flag for Review; Send to Human Queue; General Inbox |  | ## Category Routing |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node**
   - Name: `Webhook - Incoming Ticket`
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `route-ticket`
   - Response Mode: `Using Respond to Webhook Node` / `responseNode`
   - Save the node.

3. **Add a Code node after the webhook**
   - Name: `Normalize Ticket`
   - Type: `Code`
   - Paste this logic:
     ```javascript
     const raw = $input.first().json.body;
     return {
       json: {
         ticketId: raw.ticketId || 'TKT-' + Date.now(),
         subject: (raw.subject || '').trim(),
         body: (raw.body || raw.message || '').trim().substring(0, 2000),
         email: (raw.email || '').toLowerCase().trim(),
         timestamp: new Date().toISOString()
       }
     };
     ```
   - Connect: `Webhook - Incoming Ticket -> Normalize Ticket`

4. **Add an OpenRouter Chat Model node**
   - Name: `OpenRouter Chat Model`
   - Type: `OpenRouter Chat Model`
   - Model: `google/gemini-3-flash-preview`
   - Create or select OpenRouter credentials:
     - Credential type: OpenRouter API
     - Provide your OpenRouter API key
   - This node will not connect through the normal main output line.

5. **Add an AI Agent node**
   - Name: `AI - Classify`
   - Type: `AI Agent`
   - Prompt type: `Define below`
   - Prompt text:
     ```text
     Classify this support ticket.

     Subject: {{ $json.subject }}
     Body: {{ $json.body }}

     Return ONLY valid JSON:
     {"category": "billing|technical|sales|general", "urgency": "low|normal|urgent", "confidence": 0.0 to 1.0}
     ```
   - Connect the model port:
     - `OpenRouter Chat Model -> AI - Classify` through the AI language model connection
   - Connect the main flow:
     - `Normalize Ticket -> AI - Classify`

6. **Add a Code node to parse and validate the AI output**
   - Name: `Parse + Validate`
   - Type: `Code`
   - Paste this logic:
     ```javascript
     const raw = $input.first().json;
     let parsed;
     try {
       const text = typeof raw.output === 'string' ? raw.output : JSON.stringify(raw.output);
       const jsonMatch = text.match(/\{[\s\S]*\}/);
       parsed = JSON.parse(jsonMatch ? jsonMatch[0] : text);
     } catch(e) {
       parsed = { category: 'general', urgency: 'normal', confidence: 0 };
     }
     const validCats = ['billing','technical','sales','general'];
     if (!validCats.includes(parsed.category)) parsed.category = 'general';
     if (typeof parsed.confidence !== 'number') parsed.confidence = 0;
     return {
       json: {
         ...parsed,
         ticketId: $('Normalize Ticket').first().json.ticketId,
         email: $('Normalize Ticket').first().json.email
       }
     };
     ```
   - Connect: `AI - Classify -> Parse + Validate`

7. **Add a Switch node for confidence routing**
   - Name: `Confidence Threshold`
   - Type: `Switch`
   - Create 3 outputs with renamed labels:
     - Output `High Confidence`: condition `{{$json.confidence}} >= 0.85`
     - Output `Medium Confidence`: condition `{{$json.confidence}} >= 0.6`
     - Output `Low Confidence`: condition `{{$json.confidence}} < 0.6`
   - Use numeric comparisons with strict type validation if available.
   - Keep the outputs in this order so high confidence is matched before medium confidence.
   - Connect: `Parse + Validate -> Confidence Threshold`

8. **Add a Switch node for category routing**
   - Name: `Route by Category`
   - Type: `Switch`
   - Create 3 outputs with renamed labels:
     - Output `Billing`: `{{$json.category}} == "billing"`
     - Output `Technical`: `{{$json.category}} == "technical"`
     - Output `Sales`: `{{$json.category}} == "sales"`
   - Connect the first output of `Confidence Threshold` to this node:
     - `Confidence Threshold (High Confidence) -> Route by Category`

9. **Add a Code node for medium-confidence handling**
   - Name: `Flag for Review`
   - Type: `Code`
   - Code:
     ```javascript
     return { json: { ...$input.first().json, action: 'flagged_for_review' } };
     ```
   - Connect:
     - `Confidence Threshold (Medium Confidence) -> Flag for Review`

10. **Add a Code node for low-confidence handling**
    - Name: `Send to Human Queue`
    - Type: `Code`
    - Code:
      ```javascript
      return { json: { ...$input.first().json, action: 'sent_to_human' } };
      ```
    - Connect:
      - `Confidence Threshold (Low Confidence) -> Send to Human Queue`

11. **Add a Code node for billing route**
    - Name: `Billing Team`
    - Type: `Code`
    - Code:
      ```javascript
      return { json: { ...$input.first().json, routed: 'billing_team' } };
      ```
    - Connect:
      - `Route by Category (Billing) -> Billing Team`

12. **Add a Code node for technical route**
    - Name: `Technical Team`
    - Type: `Code`
    - Code:
      ```javascript
      return { json: { ...$input.first().json, routed: 'technical_team' } };
      ```
    - Connect:
      - `Route by Category (Technical) -> Technical Team`

13. **Add a Code node for sales route**
    - Name: `Sales Team`
    - Type: `Code`
    - Code:
      ```javascript
      return { json: { ...$input.first().json, routed: 'sales_team' } };
      ```
    - Connect:
      - `Route by Category (Sales) -> Sales Team`

14. **Add a Code node for general/default routing**
    - Name: `General Inbox`
    - Type: `Code`
    - Code:
      ```javascript
      return { json: { ...$input.first().json, routed: 'general_inbox' } };
      ```
    - In the provided workflow JSON, this node exists but is not connected from `Route by Category`.
    - To reproduce the JSON exactly, leave it unconnected on input and only connect its output to Respond.
    - To make the design fully functional, add a fourth category branch for `general` or a fallback route from `Route by Category` into `General Inbox`.

15. **Add a Respond to Webhook node**
    - Name: `Respond`
    - Type: `Respond to Webhook`
    - Response format: `JSON`
    - Response body:
      ```javascript
      {{ JSON.stringify({ ticketId: $json.ticketId, routed: $json.routed || $json.action, category: $json.category, confidence: $json.confidence }) }}
      ```
    - Connect these nodes into `Respond`:
      - `Billing Team`
      - `Technical Team`
      - `Sales Team`
      - `Flag for Review`
      - `Send to Human Queue`
      - `General Inbox`

16. **Optional: add sticky notes for documentation**
    - Add one large note describing the complete flow:
      - Classification-then-Routing
      - Setup notes about LLM credentials, webhook testing, and threshold tuning
      - Customization ideas such as replacing code nodes with Slack/Jira/email integrations
    - Add section notes:
      - `Receive & Normalize`
      - `AI Classification`
      - `Confidence Routing`
      - `Category Routing`

17. **Test the webhook**
    - Use the test URL from the Webhook node
    - Send a POST payload such as:
      ```json
      {
        "ticketId": "TKT-1001",
        "subject": "Payment failed during renewal",
        "body": "My credit card was charged but the invoice still shows unpaid.",
        "email": "user@example.com"
      }
      ```
    - If your webhook expects the payload inside `body`, make sure the incoming structure matches how n8n exposes request data in your environment.

18. **Credential setup requirements**
    - Required:
      - OpenRouter API credential for the `OpenRouter Chat Model` node
    - Not required in the current design:
      - Slack, Jira, email, helpdesk, or queue credentials
    - If you replace placeholder Code nodes with real integrations, add the corresponding credentials there.

19. **Important implementation note**
    - The workflow’s current JSON has a design mismatch:
      - `Parse + Validate` allows `general`
      - `Route by Category` has no `general` branch
      - `General Inbox` exists but is not connected from the switch
    - If a high-confidence ticket is classified as `general`, the workflow may not reach `Respond`, causing the webhook request to stall or fail.
    - Recommended fix:
      - Add a fourth `general` condition in `Route by Category`, or
      - Enable a fallback/default branch and connect it to `General Inbox`

20. **Recommended hardening improvements**
    - In `Normalize Ticket`, guard against missing `body` object:
      ```javascript
      const raw = $input.first().json.body || {};
      ```
    - Validate `urgency` in `Parse + Validate`
    - Clamp confidence to the range `0..1`
    - Consider adding timeout/error handling for LLM failures
    - Replace placeholder route nodes with real delivery actions

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Classification-then-Routing: Webhook receives a support ticket, Normalize Ticket cleans it, AI classifies it, Parse + Validate checks result quality, Confidence Threshold decides autonomy level, and Route by Category dispatches high-confidence tickets. | Workflow-level design note |
| Connect your LLM credentials, such as OpenRouter or OpenAI, to the Chat Model node. | Setup guidance |
| Copy the Webhook test URL and send test tickets with varying complexity to observe confidence-based routing behavior. | Testing guidance |
| Adjust confidence thresholds in the Confidence Threshold switch to match operational risk tolerance. | Tuning guidance |
| Replace team handler Code nodes with real integrations such as Slack, Jira, or email. | Customization guidance |
| Lower thresholds over time if AI accuracy proves reliable and you want more automation. | Operational optimization |