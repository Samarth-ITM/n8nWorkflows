Guardrail AI inputs and outputs with GPT-4o-mini and n8n Guardrails

https://n8nworkflows.xyz/workflows/guardrail-ai-inputs-and-outputs-with-gpt-4o-mini-and-n8n-guardrails-13853


# Guardrail AI inputs and outputs with GPT-4o-mini and n8n Guardrails

# 1. Workflow Overview

This workflow implements a guarded AI response pipeline for inbound user messages. It receives text through a webhook, extracts the relevant message content, checks the input for risky content, generates a customer-support-style response with GPT-4o-mini, then scans the generated output before returning either the safe response or a fallback message.

Its main use cases are:

- Protecting AI-powered support endpoints from unsafe user prompts
- Blocking sensitive or manipulative input before it reaches the model
- Preventing unsafe or policy-violating AI output from reaching end users
- Returning predictable JSON responses for safe, blocked, or fallback scenarios

The workflow is logically divided into the following blocks:

## 1.1 Input Reception and Text Extraction
The workflow starts with an HTTP POST webhook and normalizes the incoming payload into a single `text` field plus a `source` field.

## 1.2 Input Guardrails
The extracted text is scanned with n8n Guardrails using an attached OpenAI chat model. If the input is flagged, the workflow immediately returns a blocked response.

## 1.3 AI Response Generation
If the input passes validation, an AI Agent uses GPT-4o-mini to generate a concise customer support response under explicit behavioral constraints.

## 1.4 Output Guardrails and Final Response
The generated AI output is scanned again for unsafe content such as secret leakage or NSFW material. Safe output is returned to the caller; flagged output is replaced with a fallback message.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Text Extraction

### Overview
This block receives the HTTP request and converts varying input payload shapes into a normalized internal format. It ensures downstream nodes always work with a single `text` field regardless of whether the caller sent `message`, `body`, or `text`.

### Nodes Involved
- Webhook - User Input
- Extract Text

### Node Details

#### Webhook - User Input
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point that listens for incoming HTTP POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `guardrails-demo`
  - Response mode: `responseNode`
- **Key expressions or variables used:**
  - None inside the node itself
- **Input and output connections:**
  - Input: none, this is the trigger
  - Output: sends request data to `Extract Text`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Because response mode is `responseNode`, the workflow must end through one of the Respond to Webhook nodes
- **Edge cases or potential failure types:**
  - Wrong HTTP method returns webhook-level error
  - If no downstream response node executes, the request may hang or fail
  - Test vs production webhook URL confusion is common during setup
- **Sub-workflow reference:**
  - None

#### Extract Text
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that extracts message text from the webhook body.
- **Configuration choices:**
  - Reads the first input item’s `json.body`
  - Builds a normalized object:
    - `text`: from `input.message`, else `input.body`, else `input.text`, else empty string
    - `source`: from `input.source`, else `'unknown'`
  - Trims whitespace from the extracted text
- **Key expressions or variables used:**
  - Internal code logic:
    - `const input = $input.first().json.body;`
    - `const text = (input.message || input.body || input.text || '').trim();`
- **Input and output connections:**
  - Input: `Webhook - User Input`
  - Output: `Input Guardrails`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Requires JavaScript-enabled Code node support in current n8n
- **Edge cases or potential failure types:**
  - If `json.body` is absent or not an object, this code may throw depending on the actual payload shape
  - If the caller sends nested or differently named fields, `text` becomes empty
  - Empty string input is allowed through this node and will be handled downstream
- **Sub-workflow reference:**
  - None

---

## 2.2 Input Guardrails

### Overview
This block evaluates incoming user text before any AI generation occurs. It uses the Guardrails node, backed by a GPT-4o-mini model, to detect PII, jailbreak attempts, and secret key exposure patterns.

### Nodes Involved
- Input Guardrails
- Input Guardrails LLM
- Respond - Input Blocked

### Node Details

#### Input Guardrails
- **Type and technical role:** `@n8n/n8n-nodes-langchain.guardrails`  
  Safety screening node that classifies the input text against configured guardrail checks.
- **Configuration choices:**
  - Text to evaluate: `={{ $json.text }}`
  - Enabled checks:
    - PII: all supported PII types
    - Jailbreak detection: threshold `0.8`
    - Secret key detection: permissiveness `strict`
  - The node has two outputs:
    - Primary pass path for safe input
    - Flagged/rejected path for unsafe input
- **Key expressions or variables used:**
  - `{{$json.text}}`
- **Input and output connections:**
  - Main input: `Extract Text`
  - Main output 0: `AI - Generate Response`
  - Main output 1: `Respond - Input Blocked`
  - AI language model input: `Input Guardrails LLM`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires compatible n8n LangChain/AI nodes package with Guardrails support
- **Edge cases or potential failure types:**
  - Missing attached language model can prevent execution
  - False positives on PII or jailbreak detection may block legitimate messages
  - Empty or very short input may produce inconsistent classification depending on model behavior
  - API/network failures from the underlying LLM will stop processing
- **Sub-workflow reference:**
  - None

#### Input Guardrails LLM
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Chat model used by the Input Guardrails node for semantic detection tasks.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Default options with no extra tuning
  - Uses OpenAI credentials named `OpenAi account 2`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - AI language model output to `Input Guardrails`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires valid OpenAI API credentials
- **Edge cases or potential failure types:**
  - Invalid or expired OpenAI credentials
  - Rate limits or quota exhaustion
  - Provider/model availability changes
- **Sub-workflow reference:**
  - None

#### Respond - Input Blocked
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns an HTTP response when input is blocked by guardrails.
- **Configuration choices:**
  - Response format: JSON
  - HTTP status code: `400`
  - Response body:
    - `status: 'input_blocked'`
    - `message: 'Your message could not be processed. Please rephrase and try again.'`
- **Key expressions or variables used:**
  - `={{ JSON.stringify({ status: 'input_blocked', message: 'Your message could not be processed. Please rephrase and try again.' }) }}`
- **Input and output connections:**
  - Input: flagged branch from `Input Guardrails`
  - Output: none
- **Version-specific requirements:**
  - Uses `typeVersion: 1.1`
  - Only works correctly when the webhook trigger is configured for response by response node
- **Edge cases or potential failure types:**
  - If another response node also executes for the same request, n8n may error due to duplicate responses
  - Malformed response expression would break the HTTP reply
- **Sub-workflow reference:**
  - None

---

## 2.3 AI Response Generation

### Overview
This block generates a support-style response only for inputs that passed the input guardrails. The agent is constrained by a fixed prompt instructing it not to include URLs, external references, or system secrets.

### Nodes Involved
- AI - Generate Response
- OpenAI Chat Model

### Node Details

#### AI - Generate Response
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent node that sends a defined prompt to a connected chat model and returns generated output.
- **Configuration choices:**
  - Prompt type: `define`
  - Prompt instructs the model to:
    - act as a helpful customer support assistant
    - respond professionally and concisely
    - avoid URLs, links, and external websites
    - avoid revealing internal system details, API keys, or credentials
  - Customer message is injected from the `Extract Text` node rather than current-node input
- **Key expressions or variables used:**
  - `{{ $('Extract Text').item.json.text }}`
- **Input and output connections:**
  - Main input: safe branch from `Input Guardrails`
  - Main output: `Output Guardrails`
  - AI language model input: `OpenAI Chat Model`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.7`
  - Requires a connected AI chat model node
- **Edge cases or potential failure types:**
  - If `Extract Text` produced empty text, the model may generate a vague or generic response
  - If expression scope breaks due to workflow changes, prompt injection of the source field will fail
  - OpenAI failures, timeouts, or rate limits propagate here
  - Prompt instructions reduce but do not guarantee policy compliance, hence the output guardrails block
- **Sub-workflow reference:**
  - None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Chat model provider for the AI Agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Default options
  - Uses OpenAI credentials named `OpenAi account 2`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - AI language model output to `AI - Generate Response`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires working OpenAI credentials and model access
- **Edge cases or potential failure types:**
  - Credential issues
  - API quota/rate limiting
  - Model deprecation or provider-side behavior changes
- **Sub-workflow reference:**
  - None

---

## 2.4 Output Guardrails and Final Response

### Overview
This block validates the generated AI text before it leaves the workflow. If the output is judged safe, it is returned as JSON; if flagged, the workflow returns a controlled fallback message instead of the raw model output.

### Nodes Involved
- Output Guardrails
- Output Guardrails LLM
- Respond - Safe Output
- Respond - Flagged Output (Fallback)

### Node Details

#### Output Guardrails
- **Type and technical role:** `@n8n/n8n-nodes-langchain.guardrails`  
  Post-generation safety filter for the AI response.
- **Configuration choices:**
  - Text to evaluate: `={{ $json.output }}`
  - Enabled checks:
    - NSFW detection with threshold `0.8`
    - Secret key detection with `strict` permissiveness
  - Two output branches:
    - Safe output
    - Flagged output
- **Key expressions or variables used:**
  - `{{$json.output}}`
- **Input and output connections:**
  - Main input: `AI - Generate Response`
  - Main output 0: `Respond - Safe Output`
  - Main output 1: `Respond - Flagged Output (Fallback)`
  - AI language model input: `Output Guardrails LLM`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires compatible n8n Guardrails support and connected language model
- **Edge cases or potential failure types:**
  - False positives could suppress acceptable AI answers
  - If AI Agent output schema changes and `output` is absent, expression evaluation may fail or scan empty text
  - Upstream model errors prevent this node from receiving data
- **Sub-workflow reference:**
  - None

#### Output Guardrails LLM
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Chat model used by the output guardrails classifier.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Default options
  - Uses OpenAI credentials named `OpenAi account 2`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - AI language model output to `Output Guardrails`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires valid OpenAI access
- **Edge cases or potential failure types:**
  - Credential and quota issues
  - Model unavailability or API latency
- **Sub-workflow reference:**
  - None

#### Respond - Safe Output
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns the approved AI response to the original HTTP caller.
- **Configuration choices:**
  - Response format: JSON
  - Response body includes:
    - `status: 'success'`
    - `response: $('AI - Generate Response').item.json.output`
  - Uses explicit `JSON.stringify(...)`
- **Key expressions or variables used:**
  - `={{ JSON.stringify({ status: 'success', response: $('AI - Generate Response').item.json.output }) }}`
- **Input and output connections:**
  - Input: safe branch from `Output Guardrails`
  - Output: none
- **Version-specific requirements:**
  - Uses `typeVersion: 1.1`
  - Must be paired with webhook response mode `responseNode`
- **Edge cases or potential failure types:**
  - If the referenced `AI - Generate Response` item is unavailable, expression resolution may fail
  - Returning a stringified JSON body may differ from expectations if downstream expects raw object handling
- **Sub-workflow reference:**
  - None

#### Respond - Flagged Output (Fallback)
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns a safe fallback message when generated output is flagged.
- **Configuration choices:**
  - Response format: JSON
  - Response body includes:
    - `status: 'output_flagged'`
    - `fallback: 'Thank you for your message. A team member will follow up shortly.'`
- **Key expressions or variables used:**
  - `={{ JSON.stringify({ status: 'output_flagged', fallback: 'Thank you for your message. A team member will follow up shortly.' }) }}`
- **Input and output connections:**
  - Input: flagged branch from `Output Guardrails`
  - Output: none
- **Version-specific requirements:**
  - Uses `typeVersion: 1.1`
  - Requires webhook response mode `responseNode`
- **Edge cases or potential failure types:**
  - Duplicate response conflicts if another Respond node executes for the same request
  - Expression formatting errors would break the HTTP reply
- **Sub-workflow reference:**
  - None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - User Input | n8n-nodes-base.webhook | Receives POST requests containing user input |  | Extract Text | ## Receive & Extract |
| Extract Text | n8n-nodes-base.code | Normalizes incoming payload into `text` and `source` fields | Webhook - User Input | Input Guardrails | ## Receive & Extract |
| Input Guardrails | @n8n/n8n-nodes-langchain.guardrails | Screens inbound text for PII, jailbreak attempts, and secret-like content | Extract Text; Input Guardrails LLM | AI - Generate Response; Respond - Input Blocked | ## Input Guardrails |
| Input Guardrails LLM | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini as the language model for input safety checks |  | Input Guardrails | ## Input Guardrails |
| AI - Generate Response | @n8n/n8n-nodes-langchain.agent | Generates a customer support reply for approved input | Input Guardrails; OpenAI Chat Model | Output Guardrails | ## AI Generation |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini to the AI Agent |  | AI - Generate Response | ## AI Generation |
| Output Guardrails | @n8n/n8n-nodes-langchain.guardrails | Screens generated output for unsafe content before returning it | AI - Generate Response; Output Guardrails LLM | Respond - Safe Output; Respond - Flagged Output (Fallback) | ## Output Guardrails |
| Output Guardrails LLM | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini as the language model for output safety checks |  | Output Guardrails | ## Output Guardrails |
| Respond - Safe Output | n8n-nodes-base.respondToWebhook | Returns safe AI output as JSON | Output Guardrails |  | ## Output Guardrails |
| Respond - Flagged Output (Fallback) | n8n-nodes-base.respondToWebhook | Returns a fallback message when output is flagged | Output Guardrails |  | ## Output Guardrails |
| Respond - Input Blocked | n8n-nodes-base.respondToWebhook | Returns a 400 JSON response when the input is rejected | Input Guardrails |  |  |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation note |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace section label |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Workspace section label |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Workspace section label |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Workspace section label |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node** named **`Webhook - User Input`**.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `guardrails-demo`
   - Response Mode: `Using Respond to Webhook Node` / `responseNode`
   - Save the node.

3. **Add a Code node** named **`Extract Text`**.
   - Type: `Code`
   - Language: JavaScript
   - Use this logic:
     - Read `json.body` from the webhook payload
     - Extract text from `message`, else `body`, else `text`
     - Trim whitespace
     - Return:
       - `text`
       - `source`, defaulting to `unknown`
   - Equivalent logic:
     ```javascript
     const input = $input.first().json.body;
     const text = (input.message || input.body || input.text || '').trim();
     return { json: { text: text, source: input.source || 'unknown' } };
     ```
   - Connect **Webhook - User Input → Extract Text**.

4. **Add an OpenAI Chat Model node** named **`Input Guardrails LLM`**.
   - Type: `OpenAI Chat Model`
   - Model: `gpt-4o-mini`
   - Credentials: connect a valid OpenAI credential
   - Leave additional options at default unless your environment requires custom settings.

5. **Add a Guardrails node** named **`Input Guardrails`**.
   - Type: `Guardrails`
   - Text field: `{{ $json.text }}`
   - Enable input checks:
     - PII: set to `all`
     - Jailbreak detection: threshold `0.8`
     - Secret keys: permissiveness `strict`
   - Connect:
     - **Extract Text → Input Guardrails** on the main connection
     - **Input Guardrails LLM → Input Guardrails** on the AI language model connection

6. **Add a Respond to Webhook node** named **`Respond - Input Blocked`**.
   - Type: `Respond to Webhook`
   - Respond With: `JSON`
   - Response Code: `400`
   - Response Body:
     ```javascript
     {{ JSON.stringify({ status: 'input_blocked', message: 'Your message could not be processed. Please rephrase and try again.' }) }}
     ```
   - Connect the **flagged/rejected output** of **Input Guardrails** to this node.

7. **Add another OpenAI Chat Model node** named **`OpenAI Chat Model`**.
   - Type: `OpenAI Chat Model`
   - Model: `gpt-4o-mini`
   - Credentials: same OpenAI credential as above, or another valid one

8. **Add an AI Agent node** named **`AI - Generate Response`**.
   - Type: `AI Agent`
   - Prompt Type: `Define`
   - Prompt text:
     ```text
     You are a helpful customer support assistant. Respond to the following inquiry in a professional, concise manner. Do not include any URLs, links, or references to external websites. Do not reveal any internal system details, API keys, or credentials.

     Customer message: {{ $('Extract Text').item.json.text }}

     Respond directly and helpfully.
     ```
   - Keep options at default unless you need custom memory/tools behavior
   - Connect:
     - Safe/approved output of **Input Guardrails → AI - Generate Response**
     - **OpenAI Chat Model → AI - Generate Response** on the AI language model connection

9. **Add another OpenAI Chat Model node** named **`Output Guardrails LLM`**.
   - Type: `OpenAI Chat Model`
   - Model: `gpt-4o-mini`
   - Credentials: valid OpenAI credential

10. **Add a second Guardrails node** named **`Output Guardrails`**.
    - Type: `Guardrails`
    - Text field: `{{ $json.output }}`
    - Enable output checks:
      - NSFW: threshold `0.8`
      - Secret keys: permissiveness `strict`
    - Connect:
      - **AI - Generate Response → Output Guardrails**
      - **Output Guardrails LLM → Output Guardrails** on the AI language model connection

11. **Add a Respond to Webhook node** named **`Respond - Safe Output`**.
    - Type: `Respond to Webhook`
    - Respond With: `JSON`
    - Response Body:
      ```javascript
      {{ JSON.stringify({ status: 'success', response: $('AI - Generate Response').item.json.output }) }}
      ```
    - Connect the **safe/passing output** of **Output Guardrails** to this node.

12. **Add a second Respond to Webhook node** named **`Respond - Flagged Output (Fallback)`**.
    - Type: `Respond to Webhook`
    - Respond With: `JSON`
    - Response Body:
      ```javascript
      {{ JSON.stringify({ status: 'output_flagged', fallback: 'Thank you for your message. A team member will follow up shortly.' }) }}
      ```
    - Connect the **flagged output** of **Output Guardrails** to this node.

13. **Add optional sticky notes** for workspace clarity.
    - General overview note content:
      - Explain the input and output guardrail flow
      - Mention credential setup
      - Suggest testing with PII and prompt injection attempts
      - Mention guardrail customization points
    - Section notes:
      - `Receive & Extract`
      - `Input Guardrails`
      - `AI Generation`
      - `Output Guardrails`

14. **Configure credentials**.
    - Create or select an **OpenAI API credential**
    - Attach it to:
      - `Input Guardrails LLM`
      - `OpenAI Chat Model`
      - `Output Guardrails LLM`

15. **Test the webhook** using the test URL.
    - Send a POST request with a JSON body such as:
      ```json
      {
        "message": "I need help with my billing issue",
        "source": "web"
      }
      ```
    - Expected success response:
      - `status: success`
      - `response: ...`

16. **Test blocked input behavior**.
    - Try input containing:
      - obvious prompt injection language like “ignore your instructions”
      - fake or real-looking PII
      - secret-like keys
    - Expected response:
      - HTTP 400
      - `status: input_blocked`

17. **Test output fallback behavior**.
    - Use prompts likely to provoke disallowed content or secret-like text
    - If output guardrails trigger, expected response:
      - `status: output_flagged`
      - fallback message instead of raw AI text

18. **Activate the workflow** and switch callers to the production webhook URL when ready.

### Expected Input and Output Shape

#### Example inbound request body
The workflow expects the useful content inside the webhook `body`, with one of these fields:
- `message`
- `body`
- `text`

Example:
```json
{
  "message": "Can you help me update my account details?",
  "source": "web"
}
```

#### Internal normalized object after extraction
```json
{
  "text": "Can you help me update my account details?",
  "source": "web"
}
```

#### Possible response shapes
- **Safe response**
  ```json
  {
    "status": "success",
    "response": "..."
  }
  ```
- **Blocked input**
  ```json
  {
    "status": "input_blocked",
    "message": "Your message could not be processed. Please rephrase and try again."
  }
  ```
- **Flagged output fallback**
  ```json
  {
    "status": "output_flagged",
    "fallback": "Thank you for your message. A team member will follow up shortly."
  }
  ```

### Important Rebuild Notes
- The Guardrails nodes require attached AI language model connections; they are not standalone classifiers.
- The AI Agent also requires a connected language model node.
- Because the webhook uses `responseNode` mode, exactly one response path must complete per request.
- The current extraction logic assumes `json.body` exists. If your clients send raw JSON in another shape, adjust the Code node accordingly.
- The workflow contains **no sub-workflows** and **no Execute Workflow nodes**.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Input + Output Guardrails — How it works: 1. Webhook receives user input via POST request. 2. Extract Text pulls the message content from the payload. 3. Input Guardrails checks for blocked keywords, prompt injection patterns, and PII. Flagged inputs are blocked and return an error response. 4. AI - Generate Response processes clean inputs and generates a response. 5. Output Guardrails scans the AI response for unexpected URLs, leaked secrets, and content policy violations. Flagged outputs fall back to a safe templated response. | Workspace documentation |
| Setup — Connect OpenAI credentials (or another provider) to both Guardrails LLM nodes and the AI Agent's Chat Model. Copy the webhook test URL and send test inputs, including some with PII (for example fake SSNs) or injection attempts (for example “ignore your instructions”) to see guardrails in action. Review the Guardrails node configurations to customize which checks are active. | Workspace documentation |
| Customization — Add custom keyword lists to the Input Guardrails for your domain (for example competitor names, profanity). Adjust the Output Guardrails to enforce your specific content policies. | Workspace documentation |