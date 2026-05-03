Execute forex trading signals from Telegram on MetaTrader 5 with Claude

https://n8nworkflows.xyz/workflows/execute-forex-trading-signals-from-telegram-on-metatrader-5-with-claude-14235


# Execute forex trading signals from Telegram on MetaTrader 5 with Claude

# 1. Workflow Overview

This workflow receives Telegram trading messages through a webhook, extracts the usable message text, asks Claude via Anthropic to determine whether the message is a valid forex trading signal, parses the AI output into structured data, and forwards valid signals to an external MetaTrader 5 handler.

Its main use case is automated trade execution from forwarded Telegram messages, with a safety layer that skips empty or non-signal messages instead of blindly sending everything to MT5.

## 1.1 Input Reception

The workflow starts with a webhook that accepts inbound data from a Telegram forwarding service or custom forwarder. The next code node extracts the relevant message content from the incoming payload.

## 1.2 Message Validation

An IF node checks whether extracted text actually exists. Empty or malformed submissions are immediately acknowledged with a webhook response, preventing unnecessary AI calls.

## 1.3 AI Signal Interpretation

If text is present, the workflow sends it to a LangChain LLM chain using the Anthropic chat model and a structured output parser. This block is responsible for deciding whether the message is a trade signal and converting it into machine-readable output.

## 1.4 Output Parsing and Signal Decision

A code node processes the LLM response into a format the workflow can test reliably. Another IF node determines whether the parsed result qualifies as a valid trading signal.

## 1.5 MT5 Dispatch and Final Response

If the parsed output is a valid signal, the workflow sends it to an external HTTP endpoint that likely acts as the MT5 execution bridge. The workflow then returns a success response to the original webhook caller. Non-signals receive a skip response instead.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview

This block receives the inbound request from the Telegram forwarding layer and extracts the raw message text or relevant payload fields needed by downstream logic. It is the workflow’s public entry point.

### Nodes Involved

- Receive from Forwarder
- Extract Message

### Node Details

#### Receive from Forwarder

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Public HTTP entry point for the workflow.
- **Configuration choices:**  
  The webhook uses the ID `telegram-signal`, which gives the endpoint a stable path component in n8n. No additional parameters are shown in the JSON, so it likely accepts default POST payloads.
- **Key expressions or variables used:**  
  None explicitly shown.
- **Input and output connections:**  
  - No input; this is an entry node.
  - Outputs to **Extract Message**.
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Wrong HTTP method from sender
  - Invalid or unexpected request body structure
  - Webhook path not activated in production
  - Forwarder timing out if the workflow takes too long to answer
- **Sub-workflow reference:**  
  None.

#### Extract Message

- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript transformation node intended to normalize the incoming webhook payload and extract the Telegram message text.
- **Configuration choices:**  
  The JSON does not include the code body, so the exact logic is not visible. Based on the workflow name and downstream dependency, this node most likely reads incoming Telegram-forwarder fields and emits a `text`-like property for later checks.
- **Key expressions or variables used:**  
  Not visible in the JSON. Likely references `$json.body`, `$json.message`, or similar incoming webhook payload paths.
- **Input and output connections:**  
  - Input from **Receive from Forwarder**
  - Output to **Has text?**
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Referencing a missing property in the webhook payload
  - Returning no items
  - Producing text in an unexpected field name, causing the IF node to mis-evaluate
  - Runtime errors in custom JavaScript
- **Sub-workflow reference:**  
  None.

---

## Block 2 — Message Validation

### Overview

This block ensures the workflow only uses the AI model when meaningful text has been extracted. Empty or unusable inbound messages are short-circuited and acknowledged immediately.

### Nodes Involved

- Has text?
- Respond (Empty)

### Node Details

#### Has text?

- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branching based on whether extracted text exists.
- **Configuration choices:**  
  Parameters are not included in the JSON, but the node clearly separates:
  - **true branch:** continue to AI analysis
  - **false branch:** respond with empty/skipped outcome
- **Key expressions or variables used:**  
  Not visible. Likely checks a field such as `{{$json.text}}` for non-empty content.
- **Input and output connections:**  
  - Input from **Extract Message**
  - True output to **Analyze Signal**
  - False output to **Respond (Empty)**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - Text field exists but contains only whitespace
  - Condition checks wrong property name
  - Multiline or unusual Telegram content not considered valid due to strict condition setup
- **Sub-workflow reference:**  
  None.

#### Respond (Empty)

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends an immediate HTTP response when no usable message text is available.
- **Configuration choices:**  
  Parameters are empty in the JSON, so the response likely uses default behavior unless configured elsewhere in the editor.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input from **Has text?** false branch
  - No downstream node
- **Version-specific requirements:**  
  Uses `typeVersion: 1.1`.
- **Edge cases or potential failure types:**  
  - If the webhook node is not configured for response handling compatible with Respond to Webhook
  - Empty/default response may not be descriptive enough for the caller
- **Sub-workflow reference:**  
  None.

---

## Block 3 — AI Signal Interpretation

### Overview

This block submits the extracted Telegram text to Claude through a LangChain LLM chain and expects a structured response. It is the workflow’s decision engine for identifying actionable forex trade signals.

### Nodes Involved

- Analyze Signal
- Anthropic Chat Model
- Structured Output Parser

### Node Details

#### Analyze Signal

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  LangChain LLM chain node that orchestrates prompt execution, model invocation, and structured parsing.
- **Configuration choices:**  
  The node is connected to:
  - **Anthropic Chat Model** as its language model
  - **Structured Output Parser** as its output parser  
  This indicates the response is expected in a deterministic schema rather than free-form prose.
- **Key expressions or variables used:**  
  Not shown in the JSON. Likely passes extracted message text into the chain prompt.
- **Input and output connections:**  
  - Main input from **Has text?** true branch
  - AI language model input from **Anthropic Chat Model**
  - AI output parser input from **Structured Output Parser**
  - Main output to **Parse LLM Response**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.4`.  
  Requires n8n version with LangChain support and compatible AI connector behavior.
- **Edge cases or potential failure types:**  
  - Anthropic credential/authentication error
  - Model timeout or rate limit
  - Prompt too long for model token limits
  - Parser mismatch if the model does not respect the required schema
  - Unexpected non-JSON output if structured prompting is weak
- **Sub-workflow reference:**  
  None.

#### Anthropic Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Claude model provider node used by the chain.
- **Configuration choices:**  
  Parameters are not shown, but this node must be configured with Anthropic credentials and a model selection such as a Claude chat-capable model.
- **Key expressions or variables used:**  
  None visible in the JSON.
- **Input and output connections:**  
  - Provides `ai_languageModel` connection to **Analyze Signal**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.3`.
- **Edge cases or potential failure types:**  
  - Invalid Anthropic API key
  - Chosen model unavailable in the account or region
  - API quota exhaustion
  - Slow responses causing webhook latency issues
- **Sub-workflow reference:**  
  None.

#### Structured Output Parser

- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a structured schema on the LLM output.
- **Configuration choices:**  
  Parameters are not shown, but this node should define the expected fields for a trading signal, likely including whether the message is a signal and any parsed trade parameters.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Provides `ai_outputParser` connection to **Analyze Signal**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`.
- **Edge cases or potential failure types:**  
  - Schema too strict for the model’s output consistency
  - Missing required fields causing parser failure
  - Type mismatch, such as numeric price expected but text returned
- **Sub-workflow reference:**  
  None.

---

## Block 4 — Output Parsing and Signal Decision

### Overview

This block converts the AI result into workflow-ready fields and decides whether the message should trigger trade execution. It serves as the final gate before external trading infrastructure is called.

### Nodes Involved

- Parse LLM Response
- Is Signal?

### Node Details

#### Parse LLM Response

- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript post-processing node that likely reshapes the structured LLM output into fields suitable for HTTP dispatch and IF evaluation.
- **Configuration choices:**  
  The code itself is not visible. It probably flattens nested parser output, normalizes booleans, and extracts fields like symbol, side, entry, stop loss, and take profit.
- **Key expressions or variables used:**  
  Not visible in the JSON.
- **Input and output connections:**  
  - Input from **Analyze Signal**
  - Output to **Is Signal?**
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - LLM response shape differs from expected schema
  - Boolean/string conversion issues, e.g. `"true"` vs `true`
  - Missing fields causing undefined-property access
  - Runtime JS errors in parsing logic
- **Sub-workflow reference:**  
  None.

#### Is Signal?

- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the parsed result is a valid forex signal.
- **Configuration choices:**  
  Parameters are omitted, but the first branch leads to MT5 dispatch and the second branch to a skip response.
- **Key expressions or variables used:**  
  Not visible. Likely checks a boolean such as `{{$json.isSignal}}`.
- **Input and output connections:**  
  - Input from **Parse LLM Response**
  - True output to **Send to MT5 Handler**
  - False output to **Respond (Skipped)**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.2`.
- **Edge cases or potential failure types:**  
  - Wrong property name checked
  - Truthy string values causing unintended branch behavior
  - Partial signals passing validation when they should be rejected
- **Sub-workflow reference:**  
  None.

---

## Block 5 — MT5 Dispatch and Final Response

### Overview

This block forwards valid trade signals to the external MetaTrader 5 handler and returns a success response. If the signal is invalid or non-actionable, it returns a skip response instead.

### Nodes Involved

- Send to MT5 Handler
- Respond (Signal Sent)
- Respond (Skipped)

### Node Details

#### Send to MT5 Handler

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the parsed signal payload to an external HTTP service that likely bridges n8n with MetaTrader 5.
- **Configuration choices:**  
  Parameters are not shown. This node must define:
  - Target URL of the MT5 handler
  - HTTP method
  - Request body format
  - Authentication if required
- **Key expressions or variables used:**  
  Not visible. It likely maps parsed trade fields from the previous code node into JSON.
- **Input and output connections:**  
  - Input from **Is Signal?** true branch
  - Output to **Respond (Signal Sent)**
- **Version-specific requirements:**  
  Uses `typeVersion: 4.2`.
- **Edge cases or potential failure types:**  
  - MT5 bridge unavailable or timing out
  - Wrong payload schema
  - Authentication rejection
  - Network/firewall restrictions
  - Duplicate trade execution if retries are enabled without idempotency
- **Sub-workflow reference:**  
  None.

#### Respond (Signal Sent)

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns an HTTP response to the original webhook caller after successful MT5 handler submission.
- **Configuration choices:**  
  Parameters are not shown. It may return handler response data or a simple success message.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input from **Send to MT5 Handler**
  - No downstream node
- **Version-specific requirements:**  
  Uses `typeVersion: 1.1`.
- **Edge cases or potential failure types:**  
  - If the HTTP request succeeds but response formatting is inconsistent
  - If the webhook response is attempted after timeout
- **Sub-workflow reference:**  
  None.

#### Respond (Skipped)

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns a non-error acknowledgement for messages determined not to be valid signals.
- **Configuration choices:**  
  Parameters are empty in the JSON, so the response body likely needs manual definition in the actual implementation.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input from **Is Signal?** false branch
  - No downstream node
- **Version-specific requirements:**  
  Uses `typeVersion: 1.1`.
- **Edge cases or potential failure types:**  
  - Ambiguous response body may make debugging difficult
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Instructions1 | Sticky Note | Canvas annotation |  |  |  |
| Sticky Note | Sticky Note | Canvas annotation |  |  |  |
| Sticky Note1 | Sticky Note | Canvas annotation |  |  |  |
| Sticky Note2 | Sticky Note | Canvas annotation |  |  |  |
| Receive from Forwarder | Webhook | Receives inbound Telegram-forwarder payload |  | Extract Message |  |
| Extract Message | Code | Extracts/normalizes message text from webhook payload | Receive from Forwarder | Has text? |  |
| Has text? | If | Checks whether extracted message text exists | Extract Message | Analyze Signal; Respond (Empty) |  |
| Analyze Signal | LangChain LLM Chain | Sends message to Claude and expects structured trading-signal output | Has text? | Parse LLM Response |  |
| Respond (Empty) | Respond to Webhook | Returns response for empty or missing text | Has text? |  |  |
| Parse LLM Response | Code | Reshapes LLM output into fields usable by workflow logic | Analyze Signal | Is Signal? |  |
| Is Signal? | If | Determines whether parsed result is a valid signal | Parse LLM Response | Send to MT5 Handler; Respond (Skipped) |  |
| Send to MT5 Handler | HTTP Request | Sends valid trading signal to external MT5 bridge/handler | Is Signal? | Respond (Signal Sent) |  |
| Respond (Signal Sent) | Respond to Webhook | Returns success response after handler dispatch | Send to MT5 Handler |  |  |
| Respond (Skipped) | Respond to Webhook | Returns acknowledgement for non-signal messages | Is Signal? |  |  |
| Anthropic Chat Model | Anthropic Chat Model | Claude model backend for LLM chain |  | Analyze Signal |  |
| Structured Output Parser | Structured Output Parser | Enforces structured schema on LLM output |  | Analyze Signal |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it a name similar to:  
   **Execute forex trading signals from Telegram on MetaTrader 5 with Claude**

2. **Add a Webhook node** named **Receive from Forwarder**.
   - Node type: **Webhook**
   - Set the webhook path or ID so it resolves to something like `telegram-signal`
   - Configure it to accept the HTTP method used by your Telegram forwarding service, typically `POST`
   - Decide whether the response will be handled by a later **Respond to Webhook** node; if so, use the appropriate webhook response mode for deferred response behavior

3. **Add a Code node** named **Extract Message** and connect:
   - **Receive from Forwarder → Extract Message**
   - In this node, write JavaScript that reads the incoming webhook payload and extracts the Telegram message text into a consistent field, for example:
     - `text`
     - `rawMessage`
     - `chatId`
     - `source`
   - At minimum, ensure the node outputs one item with a `text` field containing the message body to analyze

4. **Add an IF node** named **Has text?** and connect:
   - **Extract Message → Has text?**
   - Configure the condition to verify that the extracted text field exists and is not empty
   - Best practice: trim whitespace before deciding
   - Example logical intent:
     - true if `text` is present and not blank
     - false otherwise

5. **Add a Respond to Webhook node** named **Respond (Empty)**.
   - Connect the **false** output of **Has text?** to this node
   - Configure a simple JSON response such as:
     - status: `ignored`
     - reason: `empty message`
   - Optionally set HTTP status code `200`

6. **Add an Anthropic Chat Model node** named **Anthropic Chat Model**.
   - Node type: **Anthropic Chat Model**
   - Create/select Anthropic credentials using an API key
   - Choose a Claude model suitable for extraction/classification tasks
   - Keep temperature low for consistency, typically near `0` or a low deterministic value

7. **Add a Structured Output Parser node** named **Structured Output Parser**.
   - Define a schema for the expected signal output
   - Recommended fields:
     - `isSignal` : boolean
     - `symbol` : string
     - `action` : string such as `buy` or `sell`
     - `entry` : number or string
     - `stopLoss` : number or string
     - `takeProfit` : array or single value
     - `confidence` : number or string
     - `reason` : string
   - Require `isSignal` at minimum
   - If your MT5 bridge expects exact field names, match them here

8. **Add a LangChain LLM Chain node** named **Analyze Signal**.
   - Connect:
     - **Has text?** true output → **Analyze Signal**
     - **Anthropic Chat Model** → **Analyze Signal** via AI language model connection
     - **Structured Output Parser** → **Analyze Signal** via AI output parser connection
   - Configure the prompt so the model:
     - reads the Telegram text
     - decides if it is a forex trading signal
     - extracts actionable fields only if present
     - returns output in the parser schema only
   - Example prompt intent:
     - Identify whether the message contains a forex trade instruction
     - Extract symbol, side, entry, SL, TP, and any relevant metadata
     - Mark `isSignal` false for commentary, news, or incomplete trade ideas

9. **Add a Code node** named **Parse LLM Response**.
   - Connect:
     - **Analyze Signal → Parse LLM Response**
   - In this node, normalize the chain output into a flat object for downstream use
   - Typical tasks:
     - Convert nested parser output to top-level fields
     - Ensure `isSignal` is a real boolean
     - Clean numeric strings
     - Prepare payload fields expected by the MT5 handler

10. **Add an IF node** named **Is Signal?**.
    - Connect:
      - **Parse LLM Response → Is Signal?**
    - Configure the condition to evaluate whether the parsed result is actionable
    - Minimum rule:
      - true when `isSignal === true`
    - Better rule:
      - true when `isSignal === true` and required fields like symbol/action are present

11. **Add an HTTP Request node** named **Send to MT5 Handler**.
    - Connect the **true** output of **Is Signal?** to this node
    - Configure:
      - HTTP method: usually `POST`
      - URL: your MT5 bridge endpoint
      - Body content type: JSON
      - Request body fields mapped from parsed signal data
    - Typical payload:
      - `symbol`
      - `action`
      - `entry`
      - `stopLoss`
      - `takeProfit`
      - `sourceText`
      - `confidence`
    - If required, configure authentication:
      - header token
      - basic auth
      - API key query/header
    - Enable timeout/retry settings carefully to avoid duplicate trade placement

12. **Add a Respond to Webhook node** named **Respond (Signal Sent)**.
    - Connect:
      - **Send to MT5 Handler → Respond (Signal Sent)**
    - Return a JSON acknowledgement, for example:
      - status: `sent`
      - destination: `mt5`
      - symbol: from parsed payload
    - You may also include the MT5 handler’s response if useful

13. **Add a Respond to Webhook node** named **Respond (Skipped)**.
    - Connect the **false** output of **Is Signal?** to this node
    - Return a JSON acknowledgement such as:
      - status: `skipped`
      - reason: `not a valid signal`

14. **Optionally add sticky notes** for documentation zones.
    - The provided workflow includes four sticky note nodes:
      - `Instructions1`
      - `Sticky Note`
      - `Sticky Note1`
      - `Sticky Note2`
    - Their content is empty in the JSON, so they are optional and purely visual

15. **Credential setup**
    - **Anthropic credential**
      - Add API key in n8n credentials
      - Ensure the selected Claude model is allowed for your account
    - **MT5 handler authentication**
      - Configure credentials directly in the HTTP Request node or with reusable n8n credentials depending on your bridge design

16. **Webhook sender setup**
    - Point your Telegram forwarder or intermediary service to the n8n production webhook URL for **Receive from Forwarder**
    - Ensure it sends message text in a consistent field structure
    - If Telegram messages include captions, forwarded metadata, or nested objects, account for that in **Extract Message**

17. **Testing sequence**
    - Send an empty payload and confirm **Respond (Empty)**
    - Send a non-trading message and confirm **Respond (Skipped)**
    - Send a valid forex signal and confirm:
      - Claude parses it correctly
      - **Is Signal?** passes
      - MT5 handler receives proper JSON
      - **Respond (Signal Sent)** returns success

18. **Hardening recommendations**
    - Add validation in **Parse LLM Response** for missing SL/TP values
    - Add duplicate-detection if Telegram channels resend the same signal
    - Add logging or data storage for auditability before dispatching to MT5
    - Consider adding a manual approval step for high-risk environments

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow contains four sticky note nodes, but all of them are empty in the provided JSON. | Visual annotations only; no documented content to preserve |
| The exact logic inside both Code nodes is not present in the JSON export. Reproduction requires implementing custom extraction and parsing logic manually. | Applies to `Extract Message` and `Parse LLM Response` |
| The exact prompt and schema configuration of the LLM chain are also not included in the provided JSON. These must be designed during rebuild based on the intended trade-signal format. | Applies to `Analyze Signal` and `Structured Output Parser` |
| The workflow depends on an external MT5 handler reachable over HTTP. That service is outside this workflow and must be implemented or available separately. | Applies to `Send to MT5 Handler` |