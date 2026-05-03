Validate AI ticket classifications with OpenRouter Gemini and schema checks

https://n8nworkflows.xyz/workflows/validate-ai-ticket-classifications-with-openrouter-gemini-and-schema-checks-13851


# Validate AI ticket classifications with OpenRouter Gemini and schema checks

# 1. Workflow Overview

This workflow receives a support ticket through an HTTP webhook, normalizes the incoming payload, asks an LLM hosted through OpenRouter to classify the ticket, validates the AI response against a strict semantic schema, and returns either a successful classification or a “needs review” response.

It is designed for support intake scenarios where AI-generated classifications must not be trusted blindly. The workflow enforces both structural and semantic checks before returning the result to the caller.

## 1.1 Input Reception and Normalization

The workflow starts with a webhook that accepts a POST request containing a support ticket. A Code node then standardizes the payload into a predictable internal format: ticket ID, subject, body, customer email, and timestamp.

## 1.2 AI Classification

The normalized ticket is sent to an AI Agent node. The agent uses an OpenRouter chat model configured for `google/gemini-3-flash-preview` and prompts it to return only JSON containing category, urgency, confidence, and summary.

## 1.3 Output Validation and Routing

A second Code node parses the AI output, extracts JSON if wrapped in text or markdown, and validates allowed values and constraints. An IF node then routes valid outputs to a success webhook response and invalid outputs to a 422 review-required response.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Normalization

### Overview

This block receives the incoming support ticket over HTTP and converts inconsistent source fields into a controlled internal schema. It also truncates the ticket body and standardizes email casing to improve prompt consistency.

### Nodes Involved

- Webhook - Receive Support Ticket
- Clean Ticket Data

### Node Details

#### 1. Webhook - Receive Support Ticket

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for the workflow. Exposes an HTTP endpoint that receives incoming support ticket payloads.
- **Configuration choices:**
  - HTTP Method: `POST`
  - Path: `classify-ticket`
  - Response Mode: `responseNode`
- **Key expressions or variables used:**
  - No expression logic in this node itself.
  - Incoming payload is expected under `$json.body` in the next node.
- **Input and output connections:**
  - Input: none; this is an entry node.
  - Output: `Clean Ticket Data`
- **Version-specific requirements:**
  - Uses webhook node `typeVersion: 2`
  - Because response mode is set to `responseNode`, the workflow must terminate via a Respond to Webhook node.
- **Edge cases or potential failure types:**
  - Wrong HTTP method will fail before workflow execution.
  - If callers send malformed JSON or an unexpected content type, payload parsing may not produce the expected `body` object.
  - If no downstream Respond node is reached, the webhook request may time out.
- **Sub-workflow reference:** none

#### 2. Clean Ticket Data

- **Type and technical role:** `n8n-nodes-base.code`  
  Preprocessing node that normalizes the incoming ticket fields.
- **Configuration choices:**
  - Reads `const raw = $input.first().json.body;`
  - Produces:
    - `ticketId`: from `raw.ticketId`, else `raw.id`, else `'UNKNOWN'`
    - `subject`: trimmed string
    - `body`: from `raw.body`, `raw.message`, or `raw.text`, trimmed and truncated to 2000 characters
    - `customerEmail`: lowercased and trimmed from `raw.email`
    - `receivedAt`: current ISO timestamp
- **Key expressions or variables used:**
  - `$input.first().json.body`
  - JavaScript fallbacks with `||`
  - `substring(0, 2000)` for body length control
- **Input and output connections:**
  - Input: `Webhook - Receive Support Ticket`
  - Output: `AI - Classify Ticket`
- **Version-specific requirements:**
  - Uses Code node `typeVersion: 2`
- **Edge cases or potential failure types:**
  - If `.json.body` is missing entirely, `raw.ticketId` access will throw because `raw` would be `undefined`.
  - The sticky note says it “strips HTML,” but the actual code does not remove HTML tags; it only trims and truncates text.
  - If source systems send email under another property name, it will be blank.
  - If subject/body are non-string values, the current code relies on falsy fallback behavior and may still behave unexpectedly.
- **Sub-workflow reference:** none

---

## Block 2 — AI Classification

### Overview

This block sends the normalized ticket to an AI agent and instructs it to classify the ticket into one category and one urgency level, assign a confidence score, and provide a short summary. The connected OpenRouter chat model supplies the language model backend.

### Nodes Involved

- AI - Classify Ticket
- OpenRouter Chat Model

### Node Details

#### 3. AI - Classify Ticket

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  An AI Agent node used here primarily as a prompt-driven classifier.
- **Configuration choices:**
  - Prompt type: `define`
  - Prompt includes:
    - Ticket ID
    - Subject
    - Body
  - Enforces exact output contract:
    - `category`: one of `billing`, `technical`, `sales`, `general`
    - `urgency`: one of `low`, `normal`, `urgent`
    - `confidence`: number from 0 to 1
    - `summary`: brief summary
  - Explicitly instructs: “Respond ONLY with valid JSON in this exact format”
- **Key expressions or variables used:**
  - `{{ $json.ticketId }}`
  - `{{ $json.subject }}`
  - `{{ $json.body }}`
- **Input and output connections:**
  - Main input: `Clean Ticket Data`
  - AI language model input: `OpenRouter Chat Model`
  - Main output: `Validate AI Output`
- **Version-specific requirements:**
  - Uses LangChain Agent node `typeVersion: 1.7`
  - Requires compatible n8n version with LangChain/AI nodes installed and enabled.
- **Edge cases or potential failure types:**
  - Model may still return extra prose or markdown despite the prompt.
  - Model may produce invalid JSON, unexpected enums, or stringified numbers.
  - LLM latency or provider-side limits may delay the webhook response.
  - If the chat model credential is invalid or unavailable, execution will fail here.
- **Sub-workflow reference:** none

#### 4. OpenRouter Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Provides the LLM backend used by the AI Agent.
- **Configuration choices:**
  - Model: `google/gemini-3-flash-preview`
  - No additional model options configured
  - Uses an OpenRouter credential named `OpenRouter account 2`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Output via `ai_languageModel` connection to `AI - Classify Ticket`
- **Version-specific requirements:**
  - Uses OpenRouter chat model node `typeVersion: 1`
  - Requires an OpenRouter API credential configured in n8n.
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model access restrictions on the OpenRouter account
  - Provider quota exhaustion or rate limiting
  - Temporary provider/model unavailability
- **Sub-workflow reference:** none

---

## Block 3 — Output Validation and Routing

### Overview

This block parses the AI response, extracts the JSON object if necessary, validates semantic correctness, and routes the result based on validity. It protects downstream systems from malformed or semantically invalid AI output.

### Nodes Involved

- Validate AI Output
- Is Output Valid?
- Respond - Classification Result
- Respond - Needs Review

### Node Details

#### 5. Validate AI Output

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses and validates the AI output before any response is sent.
- **Configuration choices:**
  - Reads first input item as `raw`
  - Attempts to parse `raw.output`
    - If `raw.output` is a string, uses it directly
    - Otherwise stringifies it
  - Extracts JSON using regex `/\{[\s\S]*\}/`
  - Runs semantic validation against:
    - Categories: `billing`, `technical`, `sales`, `general`
    - Urgencies: `low`, `normal`, `urgent`
    - Confidence must be numeric between `0` and `1`
    - Summary must exist and be `<= 500` chars
  - On parse failure, returns:
    - `isValid: false`
    - `validationErrors`
    - `rawOutput`
    - incremented `retryCount`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `raw.output`
  - `JSON.parse(...)`
  - regex extraction for embedded JSON
- **Input and output connections:**
  - Input: `AI - Classify Ticket`
  - Output: `Is Output Valid?`
- **Version-specific requirements:**
  - Uses Code node `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Regex extraction is greedy and may capture too much if multiple JSON-like objects appear.
  - If the AI returns valid JSON but with capitalized enums like `Billing`, validation fails.
  - If confidence is returned as a string such as `"0.95"`, it fails semantic validation.
  - `retryCount` is incremented only on parse failure, but no retry loop exists in the current workflow.
  - If the Agent node output field name changes from `output`, parsing will fail.
- **Sub-workflow reference:** none

#### 6. Is Output Valid?

- **Type and technical role:** `n8n-nodes-base.if`  
  Routes valid vs invalid classifications.
- **Configuration choices:**
  - Strict boolean check:
    - Left value: `={{ $json.isValid }}`
    - Condition: boolean is `true`
  - True branch -> success response
  - False branch -> review response
- **Key expressions or variables used:**
  - `{{ $json.isValid }}`
- **Input and output connections:**
  - Input: `Validate AI Output`
  - True output: `Respond - Classification Result`
  - False output: `Respond - Needs Review`
- **Version-specific requirements:**
  - Uses IF node `typeVersion: 2.2`
  - Strict type validation is enabled.
- **Edge cases or potential failure types:**
  - If `isValid` is missing or not a boolean, strict validation causes the condition to evaluate as false-path behavior rather than success.
- **Sub-workflow reference:** none

#### 7. Respond - Classification Result

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends a successful JSON response back to the original webhook caller.
- **Configuration choices:**
  - Respond with: JSON
  - Response body contains:
    - `category`
    - `urgency`
    - `confidence`
    - `summary`
  - Body is assembled with `JSON.stringify(...)`
- **Key expressions or variables used:**
  - `{{ JSON.stringify({ category: $json.category, urgency: $json.urgency, confidence: $json.confidence, summary: $json.summary }) }}`
- **Input and output connections:**
  - Input: true branch from `Is Output Valid?`
  - Output: none
- **Version-specific requirements:**
  - Uses Respond to Webhook node `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - If fields are missing despite `isValid = true`, response may still contain incomplete data.
  - Since `respondWith` is JSON, using `JSON.stringify` may create double-encoding in some implementations or reduce readability; many setups instead pass an object directly.
- **Sub-workflow reference:** none

#### 8. Respond - Needs Review

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends an error-style response indicating validation failure and the need for manual review.
- **Configuration choices:**
  - HTTP status code: `422`
  - Respond with: JSON
  - Response body includes:
    - `status: 'validation_failed'`
    - `errors`
    - `needsHumanReview: true`
- **Key expressions or variables used:**
  - `{{ JSON.stringify({ status: 'validation_failed', errors: $json.validationErrors, needsHumanReview: true }) }}`
- **Input and output connections:**
  - Input: false branch from `Is Output Valid?`
  - Output: none
- **Version-specific requirements:**
  - Uses Respond to Webhook node `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - If `validationErrors` is missing, the response may return `errors: undefined`.
  - Same note as above regarding JSON-stringified body in a JSON response mode.
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Receive Support Ticket | Webhook | Receives POST requests containing support ticket data |  | Clean Ticket Data | ## Structured Output Parsing + Validation<br>### How it works<br>1. **Webhook** receives a support ticket via POST request.<br>2. **Clean Ticket Data** (Code node) strips HTML, trims whitespace, and standardizes fields before AI processing.<br>3. **AI - Classify Ticket** (AI Agent + OpenRouter) classifies the ticket by category, urgency, and confidence score, returning structured JSON.<br>4. **Validate AI Output** (Code node) checks that the category is from a known list, urgency is valid, and confidence is between 0 and 1. Structural correctness alone is not enough; this checks semantic correctness.<br>5. **Is Output Valid?** (IF node) routes valid classifications to the success response and invalid ones to a "needs review" path.<br>### Setup<br>- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node<br>- Copy the Webhook test URL and send a POST with a JSON body containing a support ticket (fields: subject, body, customerEmail)<br>- Review the valid categories and urgency levels in the **Validate AI Output** code node and adjust for your system<br>### Customization<br>- Add retry logic by looping invalid outputs back to the AI with the validation error message<br>- Swap the Respond nodes for actual downstream actions (create Jira ticket, send Slack alert)<br>## Receive & Clean |
| Clean Ticket Data | Code | Normalizes incoming ticket fields for prompt usage | Webhook - Receive Support Ticket | AI - Classify Ticket | ## Structured Output Parsing + Validation<br>### How it works<br>1. **Webhook** receives a support ticket via POST request.<br>2. **Clean Ticket Data** (Code node) strips HTML, trims whitespace, and standardizes fields before AI processing.<br>3. **AI - Classify Ticket** (AI Agent + OpenRouter) classifies the ticket by category, urgency, and confidence score, returning structured JSON.<br>4. **Validate AI Output** (Code node) checks that the category is from a known list, urgency is valid, and confidence is between 0 and 1. Structural correctness alone is not enough; this checks semantic correctness.<br>5. **Is Output Valid?** (IF node) routes valid classifications to the success response and invalid ones to a "needs review" path.<br>### Setup<br>- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node<br>- Copy the Webhook test URL and send a POST with a JSON body containing a support ticket (fields: subject, body, customerEmail)<br>- Review the valid categories and urgency levels in the **Validate AI Output** code node and adjust for your system<br>### Customization<br>- Add retry logic by looping invalid outputs back to the AI with the validation error message<br>- Swap the Respond nodes for actual downstream actions (create Jira ticket, send Slack alert)<br>## Receive & Clean |
| AI - Classify Ticket | LangChain Agent | Prompts the LLM to classify the ticket into structured JSON | Clean Ticket Data; OpenRouter Chat Model (AI language model input) | Validate AI Output | ## Structured Output Parsing + Validation<br>### How it works<br>1. **Webhook** receives a support ticket via POST request.<br>2. **Clean Ticket Data** (Code node) strips HTML, trims whitespace, and standardizes fields before AI processing.<br>3. **AI - Classify Ticket** (AI Agent + OpenRouter) classifies the ticket by category, urgency, and confidence score, returning structured JSON.<br>4. **Validate AI Output** (Code node) checks that the category is from a known list, urgency is valid, and confidence is between 0 and 1. Structural correctness alone is not enough; this checks semantic correctness.<br>5. **Is Output Valid?** (IF node) routes valid classifications to the success response and invalid ones to a "needs review" path.<br>### Setup<br>- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node<br>- Copy the Webhook test URL and send a POST with a JSON body containing a support ticket (fields: subject, body, customerEmail)<br>- Review the valid categories and urgency levels in the **Validate AI Output** code node and adjust for your system<br>### Customization<br>- Add retry logic by looping invalid outputs back to the AI with the validation error message<br>- Swap the Respond nodes for actual downstream actions (create Jira ticket, send Slack alert)<br>## AI Classification |
| OpenRouter Chat Model | OpenRouter Chat Model | Supplies the Gemini model used by the AI Agent |  | AI - Classify Ticket | ## AI Classification |
| Validate AI Output | Code | Parses AI output and enforces semantic validation rules | AI - Classify Ticket | Is Output Valid? | ## Structured Output Parsing + Validation<br>### How it works<br>1. **Webhook** receives a support ticket via POST request.<br>2. **Clean Ticket Data** (Code node) strips HTML, trims whitespace, and standardizes fields before AI processing.<br>3. **AI - Classify Ticket** (AI Agent + OpenRouter) classifies the ticket by category, urgency, and confidence score, returning structured JSON.<br>4. **Validate AI Output** (Code node) checks that the category is from a known list, urgency is valid, and confidence is between 0 and 1. Structural correctness alone is not enough; this checks semantic correctness.<br>5. **Is Output Valid?** (IF node) routes valid classifications to the success response and invalid ones to a "needs review" path.<br>### Setup<br>- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node<br>- Copy the Webhook test URL and send a POST with a JSON body containing a support ticket (fields: subject, body, customerEmail)<br>- Review the valid categories and urgency levels in the **Validate AI Output** code node and adjust for your system<br>### Customization<br>- Add retry logic by looping invalid outputs back to the AI with the validation error message<br>- Swap the Respond nodes for actual downstream actions (create Jira ticket, send Slack alert)<br>## Validate & Route |
| Is Output Valid? | IF | Branches workflow based on validation result | Validate AI Output | Respond - Classification Result; Respond - Needs Review | ## Structured Output Parsing + Validation<br>### How it works<br>1. **Webhook** receives a support ticket via POST request.<br>2. **Clean Ticket Data** (Code node) strips HTML, trims whitespace, and standardizes fields before AI processing.<br>3. **AI - Classify Ticket** (AI Agent + OpenRouter) classifies the ticket by category, urgency, and confidence score, returning structured JSON.<br>4. **Validate AI Output** (Code node) checks that the category is from a known list, urgency is valid, and confidence is between 0 and 1. Structural correctness alone is not enough; this checks semantic correctness.<br>5. **Is Output Valid?** (IF node) routes valid classifications to the success response and invalid ones to a "needs review" path.<br>### Setup<br>- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node<br>- Copy the Webhook test URL and send a POST with a JSON body containing a support ticket (fields: subject, body, customerEmail)<br>- Review the valid categories and urgency levels in the **Validate AI Output** code node and adjust for your system<br>### Customization<br>- Add retry logic by looping invalid outputs back to the AI with the validation error message<br>- Swap the Respond nodes for actual downstream actions (create Jira ticket, send Slack alert)<br>## Validate & Route |
| Respond - Classification Result | Respond to Webhook | Returns successful classification JSON to caller | Is Output Valid? |  | ## Structured Output Parsing + Validation<br>### How it works<br>1. **Webhook** receives a support ticket via POST request.<br>2. **Clean Ticket Data** (Code node) strips HTML, trims whitespace, and standardizes fields before AI processing.<br>3. **AI - Classify Ticket** (AI Agent + OpenRouter) classifies the ticket by category, urgency, and confidence score, returning structured JSON.<br>4. **Validate AI Output** (Code node) checks that the category is from a known list, urgency is valid, and confidence is between 0 and 1. Structural correctness alone is not enough; this checks semantic correctness.<br>5. **Is Output Valid?** (IF node) routes valid classifications to the success response and invalid ones to a "needs review" path.<br>### Setup<br>- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node<br>- Copy the Webhook test URL and send a POST with a JSON body containing a support ticket (fields: subject, body, customerEmail)<br>- Review the valid categories and urgency levels in the **Validate AI Output** code node and adjust for your system<br>### Customization<br>- Add retry logic by looping invalid outputs back to the AI with the validation error message<br>- Swap the Respond nodes for actual downstream actions (create Jira ticket, send Slack alert)<br>## Validate & Route |
| Respond - Needs Review | Respond to Webhook | Returns 422 response when AI output fails validation | Is Output Valid? |  | ## Structured Output Parsing + Validation<br>### How it works<br>1. **Webhook** receives a support ticket via POST request.<br>2. **Clean Ticket Data** (Code node) strips HTML, trims whitespace, and standardizes fields before AI processing.<br>3. **AI - Classify Ticket** (AI Agent + OpenRouter) classifies the ticket by category, urgency, and confidence score, returning structured JSON.<br>4. **Validate AI Output** (Code node) checks that the category is from a known list, urgency is valid, and confidence is between 0 and 1. Structural correctness alone is not enough; this checks semantic correctness.<br>5. **Is Output Valid?** (IF node) routes valid classifications to the success response and invalid ones to a "needs review" path.<br>### Setup<br>- Connect your **LLM credentials** (e.g., OpenRouter, OpenAI) to the Chat Model node<br>- Copy the Webhook test URL and send a POST with a JSON body containing a support ticket (fields: subject, body, customerEmail)<br>- Review the valid categories and urgency levels in the **Validate AI Output** code node and adjust for your system<br>### Customization<br>- Add retry logic by looping invalid outputs back to the AI with the validation error message<br>- Swap the Respond nodes for actual downstream actions (create Jira ticket, send Slack alert)<br>## Validate & Route |
| Sticky Note | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note1 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note2 | Sticky Note | Documentation/comment node |  |  |  |
| Sticky Note3 | Sticky Note | Documentation/comment node |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node** named **`Webhook - Receive Support Ticket`**.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `classify-ticket`.
   - Set **Response Mode** to `Using Respond to Webhook Node` / `responseNode`.
   - Leave other options at default unless your environment requires authentication or CORS configuration.

3. **Add a Code node** named **`Clean Ticket Data`** and connect it after the webhook.
   - Paste this logic conceptually:
     - Read the incoming payload from the webhook body.
     - Map incoming fields into:
       - `ticketId`
       - `subject`
       - `body`
       - `customerEmail`
       - `receivedAt`
     - Apply normalization:
       - trim whitespace
       - lowercase email
       - truncate body to 2000 characters
   - Use this field source mapping:
     - `ticketId`: `ticketId` or `id` or fallback `UNKNOWN`
     - `subject`: `subject`
     - `body`: `body` or `message` or `text`
     - `customerEmail`: `email`

4. **Add an OpenRouter Chat Model node** named **`OpenRouter Chat Model`**.
   - Choose node type: OpenRouter chat model from the AI/LangChain nodes.
   - Set **Model** to `google/gemini-3-flash-preview`.
   - Create or select an **OpenRouter API credential**.
     - In credentials, provide your OpenRouter API key.
     - Ensure the account has access to the selected model.
   - Leave advanced model options empty unless you need tighter behavior control.

5. **Add an AI Agent node** named **`AI - Classify Ticket`**.
   - Set the agent/prompt mode to a direct prompt definition.
   - In the main text prompt, instruct the model to classify one support ticket and return only valid JSON.
   - Include the normalized fields using expressions:
     - `{{ $json.ticketId }}`
     - `{{ $json.subject }}`
     - `{{ $json.body }}`
   - Specify exact allowed values:
     - category: `billing`, `technical`, `sales`, `general`
     - urgency: `low`, `normal`, `urgent`
     - confidence: numeric `0–1`
     - summary: brief summary
   - Include a strict JSON example in the prompt.
   - Connect:
     - Main input from `Clean Ticket Data`
     - AI language model input from `OpenRouter Chat Model`

6. **Add a Code node** named **`Validate AI Output`** and connect it after the AI Agent.
   - Configure it to:
     - read the agent result
     - parse the `output` property
     - extract JSON if the model wraps it in text or markdown
     - validate semantic rules
   - Validation rules:
     - `category` must be one of `billing`, `technical`, `sales`, `general`
     - `urgency` must be one of `low`, `normal`, `urgent`
     - `confidence` must be a number between `0` and `1`
     - `summary` must exist and be at most `500` characters
   - On parse failure, output:
     - `isValid: false`
     - `validationErrors: [...]`
     - `rawOutput`
     - `retryCount`
   - On successful parse, output parsed fields plus:
     - `isValid`
     - `validationErrors`

7. **Add an IF node** named **`Is Output Valid?`** and connect it after `Validate AI Output`.
   - Condition type: boolean
   - Left value: `{{ $json.isValid }}`
   - Operation: `is true`
   - Keep strict type validation enabled if you want exact boolean matching.

8. **Add a Respond to Webhook node** named **`Respond - Classification Result`**.
   - Connect it to the **true** output of the IF node.
   - Set **Respond With** to `JSON`.
   - Build a response body containing:
     - `category`
     - `urgency`
     - `confidence`
     - `summary`
   - You can mirror the workflow behavior by building the body via expression and `JSON.stringify`, though returning a structured object directly is often cleaner.

9. **Add another Respond to Webhook node** named **`Respond - Needs Review`**.
   - Connect it to the **false** output of the IF node.
   - Set **Respond With** to `JSON`.
   - Set **HTTP Response Code** to `422`.
   - Build a response body containing:
     - `status: "validation_failed"`
     - `errors: $json.validationErrors`
     - `needsHumanReview: true`

10. **Add optional Sticky Notes** for clarity if you want to match the original layout.
    - One overview note explaining the full flow.
    - One note titled `Receive & Clean`
    - One note titled `AI Classification`
    - One note titled `Validate & Route`

11. **Verify connections**:
    - `Webhook - Receive Support Ticket` -> `Clean Ticket Data`
    - `Clean Ticket Data` -> `AI - Classify Ticket`
    - `OpenRouter Chat Model` -> `AI - Classify Ticket` via AI language model connector
    - `AI - Classify Ticket` -> `Validate AI Output`
    - `Validate AI Output` -> `Is Output Valid?`
    - `Is Output Valid?` true -> `Respond - Classification Result`
    - `Is Output Valid?` false -> `Respond - Needs Review`

12. **Test with a sample POST request** to the webhook test URL.
    - Example payload shape:
      ```json
      {
        "ticketId": "T-1001",
        "subject": "Charged twice for subscription",
        "body": "I was billed two times this month. Please help.",
        "email": "customer@example.com"
      }
      ```
    - Expected success: JSON classification payload
    - Expected failure: 422 response with validation errors

13. **Activate the workflow** if you want to use the production webhook URL.

## Credential Configuration

### OpenRouter API Credential

- Required by: `OpenRouter Chat Model`
- Minimum setup:
  - Valid OpenRouter API key
- Important checks:
  - The selected model must be available to the account
  - Billing/quota must be sufficient
  - Network egress from the n8n instance must allow OpenRouter access

## Input/Output Expectations

### Expected incoming webhook payload

The workflow expects a JSON body. Useful fields include:

- `ticketId` or `id`
- `subject`
- `body` or `message` or `text`
- `email`

### Success response

Typical structure:

```json
{
  "category": "billing",
  "urgency": "normal",
  "confidence": 0.95,
  "summary": "Customer reports duplicate billing for current subscription cycle."
}
```

### Validation-failure response

Typical structure:

```json
{
  "status": "validation_failed",
  "errors": [
    "Invalid category: finance"
  ],
  "needsHumanReview": true
}
```

## Sub-workflow Setup

This workflow does **not** use any Execute Workflow or sub-workflow nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow uses AI output validation as a safety layer before returning results to the caller. | General design pattern |
| The overview sticky note suggests HTML stripping in the cleaning stage, but the current Code node does not actually remove HTML tags. | Implementation discrepancy to review |
| The overview note suggests adding retry logic by feeding validation errors back into the AI node, but this is not implemented in the current workflow. | Suggested enhancement |
| The overview note also suggests replacing webhook responses with downstream operational actions such as Jira ticket creation or Slack alerts. | Suggested customization |