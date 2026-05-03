Route AI tasks with OpenAI GPT‑4.1‑mini and confidence‑based email fallback

https://n8nworkflows.xyz/workflows/route-ai-tasks-with-openai-gpt-4-1-mini-and-confidence-based-email-fallback-14683


# Route AI tasks with OpenAI GPT‑4.1‑mini and confidence‑based email fallback

# 1. Workflow Overview

This workflow receives a task request through a webhook, classifies the request with an OpenAI-powered supervisor agent, and then either:

- routes the task to an execution agent that chooses the correct specialized tool for processing, or
- sends an email alert for human review if the classification confidence is below a defined threshold.

Its main use case is AI task routing with a safety gate: simple and complex requests are handled differently, but uncertain classifications are escalated instead of being executed automatically.

## 1.1 Input Reception and Runtime Configuration
The workflow starts from a webhook and immediately sets two core runtime values:

- `userRequest`: the incoming task to analyze and process
- `confidenceThreshold`: the minimum confidence required to allow autonomous execution

## 1.2 Task Classification
A supervisor AI agent evaluates the request and returns structured JSON with:

- `complexity`: `simple` or `complex`
- `confidence`: numeric score between 0 and 1
- `reasoning`: short explanation

This block uses a structured output parser to enforce predictable AI output.

## 1.3 Confidence Gate
An IF node checks whether the supervisor’s confidence score is greater than or equal to the configured threshold.

- If yes: continue to execution
- If no: trigger fallback email alert

## 1.4 Execution Routing
If confidence is sufficient, an orchestrator AI agent receives the original user request plus the classification result and decides which specialized tool to call:

- Simple Task Agent Tool
- Complex Task Agent Tool

## 1.5 Specialized Task Processing
Each specialized tool is backed by its own OpenAI chat model and prompt:

- the simple agent is optimized for quick, direct, single-step work
- the complex agent is optimized for deeper reasoning and detailed responses

## 1.6 Human Fallback Handling
Low-confidence classifications trigger an email notification containing the original request, predicted complexity, confidence score, reasoning, and threshold value for manual review.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Runtime Configuration

### Overview
This block receives the workflow trigger and initializes the values used throughout the workflow. It establishes the original request text and the confidence threshold used later for routing decisions.

### Nodes Involved
- Webhook
- Workflow Configuration

### Node Details

#### Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; entry point for external HTTP requests
- **Configuration choices:**
  - Uses a fixed webhook path
  - `responseMode` is set to `lastNode`, so the HTTP response is based on the final node reached in the active branch
- **Key expressions or variables used:** none directly
- **Input and output connections:**
  - No input
  - Outputs to `Workflow Configuration`
- **Version-specific requirements:** typeVersion `2.1`
- **Edge cases or potential failure types:**
  - Webhook path not activated in production/test mode
  - Unexpected request payload shape if you intend to replace placeholders dynamically
  - Long-running AI branches may delay webhook response because response mode waits for the last node
- **Sub-workflow reference:** none

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`; creates/overrides workflow variables
- **Configuration choices:**
  - Sets `userRequest` as a string placeholder
  - Sets `confidenceThreshold` to `0.7`
  - `includeOtherFields` is enabled, so existing incoming fields are retained
- **Key expressions or variables used:**
  - `userRequest`
  - `confidenceThreshold`
- **Input and output connections:**
  - Input from `Webhook`
  - Output to `Supervisor Agent`
- **Version-specific requirements:** typeVersion `3.4`
- **Edge cases or potential failure types:**
  - As configured, `userRequest` is a placeholder literal, not extracted from the webhook payload
  - If not replaced, the AI will classify the placeholder text rather than real user input
  - Threshold values outside 0–1 would make the confidence gate logically invalid
- **Sub-workflow reference:** none

---

## 2.2 Task Classification

### Overview
This block classifies the incoming task as simple or complex and assigns a confidence score. It relies on a dedicated OpenAI model plus a structured output parser to keep the result machine-readable.

### Nodes Involved
- Supervisor Agent
- OpenAI Model - Supervisor
- Routing Decision Parser

### Node Details

#### Supervisor Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; LLM agent that performs classification
- **Configuration choices:**
  - Prompt input text is `{{ $json.userRequest }}`
  - Uses a custom system message instructing the agent to:
    - analyze complexity
    - classify as `simple` or `complex`
    - produce confidence from `0.0` to `1.0`
    - explain reasoning
  - `hasOutputParser` is enabled
- **Key expressions or variables used:**
  - `={{ $json.userRequest }}`
- **Input and output connections:**
  - Main input from `Workflow Configuration`
  - AI language model input from `OpenAI Model - Supervisor`
  - AI output parser input from `Routing Decision Parser`
  - Main output to `Check Confidence Score`
- **Version-specific requirements:** typeVersion `3`
- **Edge cases or potential failure types:**
  - OpenAI credential or model access errors
  - Structured output may fail if the model returns malformed JSON
  - Ambiguous requests can produce unstable classifications
  - Empty `userRequest` may result in unreliable reasoning and confidence
- **Sub-workflow reference:** none

#### OpenAI Model - Supervisor
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; chat model attached to the supervisor agent
- **Configuration choices:**
  - Model is `gpt-4.1-mini`
  - No additional options or built-in tools configured
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Connects as `ai_languageModel` to `Supervisor Agent`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases or potential failure types:**
  - Missing OpenAI credentials
  - Unsupported model availability in the account/region
  - Rate limits or API timeouts
- **Sub-workflow reference:** none

#### Routing Decision Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates and structures LLM output
- **Configuration choices:**
  - Manual JSON schema with required fields:
    - `complexity`: enum `simple` or `complex`
    - `confidence`: number between 0 and 1
    - `reasoning`: string
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Connects as `ai_outputParser` to `Supervisor Agent`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases or potential failure types:**
  - Parser failure if output is not valid against schema
  - Confidence returned as string instead of number may cause validation issues
  - Missing required properties cause the node chain to fail
- **Sub-workflow reference:** none

---

## 2.3 Confidence Gate

### Overview
This block decides whether the classification is trusted enough for automated execution. It compares the supervisor’s confidence score against the threshold configured earlier.

### Nodes Involved
- Check Confidence Score

### Node Details

#### Check Confidence Score
- **Type and technical role:** `n8n-nodes-base.if`; conditional branch controller
- **Configuration choices:**
  - Compares `{{ $json.confidence }}` against `{{ $('Workflow Configuration').first().json.confidenceThreshold }}`
  - Uses numeric `greater than or equal`
- **Key expressions or variables used:**
  - `={{ $json.confidence }}`
  - `={{ $('Workflow Configuration').first().json.confidenceThreshold }}`
- **Input and output connections:**
  - Input from `Supervisor Agent`
  - True output to `Execute Selected Agent`
  - False output to `Send Fallback Alert`
- **Version-specific requirements:** typeVersion `2.3`
- **Edge cases or potential failure types:**
  - If `confidence` is missing or not numeric, loose validation may still behave unexpectedly
  - If threshold is null or malformed, routing may fail or misroute
  - Borderline values such as exactly `0.7` pass because operation is `gte`
- **Sub-workflow reference:** none

---

## 2.4 Execution Routing

### Overview
This block runs only when the confidence check passes. It uses an orchestrator agent that receives the original request and the supervisor’s classification context, then chooses the appropriate specialized agent tool.

### Nodes Involved
- Execute Selected Agent
- OpenAI Model - Executor

### Node Details

#### Execute Selected Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrator agent that can call AI tools
- **Configuration choices:**
  - Prompt text uses the original request:
    - `{{ $('Workflow Configuration').first().json.userRequest }}`
  - System message instructs the agent to:
    - analyze the request
    - use the supervisor classification
    - call the simple or complex tool accordingly
    - return the selected tool’s result
  - The system prompt includes:
    - `{{ $json.complexity }}`
    - `{{ $json.reasoning }}`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.userRequest }}`
  - `{{ $json.complexity }}`
  - `{{ $json.reasoning }}`
- **Input and output connections:**
  - Main input from true branch of `Check Confidence Score`
  - AI language model input from `OpenAI Model - Executor`
  - AI tool inputs from:
    - `Simple Task Agent Tool`
    - `Complex Task Agent Tool`
- **Version-specific requirements:** typeVersion `3`
- **Edge cases or potential failure types:**
  - The system message begins with `=You are...`; this looks intentional for expression mode, but should be verified in the UI to ensure the whole field is treated correctly
  - If the model ignores tool-use intent, routing may be suboptimal
  - If tools are unavailable or misconfigured, the executor may fail to return a final answer
- **Sub-workflow reference:** none

#### OpenAI Model - Executor
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; chat model powering the orchestrator
- **Configuration choices:**
  - Model is `gpt-4.1-mini`
  - No custom options enabled
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Connects as `ai_languageModel` to `Execute Selected Agent`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases or potential failure types:**
  - Credential failure
  - Rate limiting under concurrent requests
  - Model/tool-calling behavior may vary across n8n or provider updates
- **Sub-workflow reference:** none

---

## 2.5 Specialized Task Processing

### Overview
This block contains the two tool-enabled specialist agents. The executor agent calls one of them based on complexity. Each tool has its own prompt and dedicated OpenAI model.

### Nodes Involved
- Simple Task Agent Tool
- OpenAI Model - Simple Agent(mini-model)
- Complex Task Agent Tool
- OpenAI Model - Complex Agent

### Node Details

#### Simple Task Agent Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; callable AI tool for straightforward tasks
- **Configuration choices:**
  - Tool input text uses AI-provided argument:
    - `{{ $fromAI("task", "The task to process") }}`
  - System message focuses on:
    - basic questions
    - simple information
    - single-step operations
    - concise responses
  - Tool description clearly states its intended usage
- **Key expressions or variables used:**
  - `={{ $fromAI("task", "The task to process") }}`
- **Input and output connections:**
  - AI language model input from `OpenAI Model - Simple Agent(mini-model)`
  - AI tool output to `Execute Selected Agent`
- **Version-specific requirements:** typeVersion `2.2`
- **Edge cases or potential failure types:**
  - If the executor passes an empty or poor `task` argument, output quality drops
  - Tool may still answer complex tasks if called incorrectly
  - Requires compatible AI tool-calling support in current n8n version
- **Sub-workflow reference:** none

#### OpenAI Model - Simple Agent(mini-model)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model for the simple task tool
- **Configuration choices:**
  - Model is `gpt-4.1-mini`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Connects as `ai_languageModel` to `Simple Task Agent Tool`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases or potential failure types:**
  - Same standard OpenAI issues: missing credentials, rate limits, timeout
- **Sub-workflow reference:** none

#### Complex Task Agent Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; callable AI tool for advanced tasks
- **Configuration choices:**
  - Tool input text uses:
    - `{{ $fromAI("task", "The task to process") }}`
  - System message emphasizes:
    - multi-step reasoning
    - in-depth analysis
    - domain expertise
    - creative problem-solving
    - detailed explanations
  - Tool description clarifies that it handles sophisticated tasks
- **Key expressions or variables used:**
  - `={{ $fromAI("task", "The task to process") }}`
- **Input and output connections:**
  - AI language model input from `OpenAI Model - Complex Agent`
  - AI tool output to `Execute Selected Agent`
- **Version-specific requirements:** typeVersion `2.2`
- **Edge cases or potential failure types:**
  - Longer reasoning may increase latency and token usage
  - If called for trivial tasks, responses may be unnecessarily verbose
  - Depends on working AI tool orchestration from the executor agent
- **Sub-workflow reference:** none

#### OpenAI Model - Complex Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model for the complex task tool
- **Configuration choices:**
  - Model is `gpt-4.1-mini`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Connects as `ai_languageModel` to `Complex Task Agent Tool`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases or potential failure types:**
  - Same OpenAI operational issues as other model nodes
- **Sub-workflow reference:** none

---

## 2.6 Human Fallback Handling

### Overview
This block handles uncertain classifications by notifying a human reviewer instead of executing the task automatically. It acts as the workflow’s safety fallback.

### Nodes Involved
- Send Fallback Alert

### Node Details

#### Send Fallback Alert
- **Type and technical role:** `n8n-nodes-base.emailSend`; sends an email notification
- **Configuration choices:**
  - Sends plain text email
  - Subject: `Low Confidence Alert: Task Classification Uncertain`
  - Includes in body:
    - original user request
    - complexity
    - confidence score
    - reasoning
    - threshold
  - Uses placeholder sender and admin recipient email addresses
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.userRequest }}`
  - `{{ $json.complexity }}`
  - `{{ $json.confidence }}`
  - `{{ $json.reasoning }}`
  - `{{ $('Workflow Configuration').first().json.confidenceThreshold }}`
- **Input and output connections:**
  - Input from false branch of `Check Confidence Score`
  - No downstream node
- **Version-specific requirements:** typeVersion `2.1`
- **Edge cases or potential failure types:**
  - SMTP/transport credentials not configured
  - Sender domain may be rejected by mail server
  - Placeholder addresses must be replaced before use
  - Since webhook response mode is `lastNode`, the HTTP caller may receive email-node output rather than a business-facing response
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | HTTP trigger entry point |  | Workflow Configuration | ## Input Layer<br>Webhook trigger + initial workflow configuration |
| Workflow Configuration | n8n-nodes-base.set | Initialize request and confidence threshold | Webhook | Supervisor Agent | ## Input Layer<br>Webhook trigger + initial workflow configuration |
| Supervisor Agent | @n8n/n8n-nodes-langchain.agent | Classify request complexity and confidence | Workflow Configuration; OpenAI Model - Supervisor; Routing Decision Parser | Check Confidence Score | ## Task Classification<br>Supervisor agent analyzes and scores complexity |
| OpenAI Model - Supervisor | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for supervisor classification |  | Supervisor Agent | ## Task Classification<br>Supervisor agent analyzes and scores complexity |
| Routing Decision Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON classification output |  | Supervisor Agent | ## Task Classification<br>Supervisor agent analyzes and scores complexity |
| Check Confidence Score | n8n-nodes-base.if | Compare confidence with threshold | Supervisor Agent | Execute Selected Agent; Send Fallback Alert | ## Confidence Check<br>Validate if classification meets threshold |
| Execute Selected Agent | @n8n/n8n-nodes-langchain.agent | Orchestrate tool selection and final task execution | Check Confidence Score; OpenAI Model - Executor; Simple Task Agent Tool; Complex Task Agent Tool |  | ## Execution Router<br>Orchestrator selects correct agent tool<br>## AI Processing Core<br>All LLM models powering agents and routing |
| OpenAI Model - Executor | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for orchestrator agent |  | Execute Selected Agent | ## AI Processing Core<br>All LLM models powering agents and routing |
| Simple Task Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Tool for simple requests | OpenAI Model - Simple Agent(mini-model) | Execute Selected Agent | ## Simple Task Flow<br>Handles quick, single-step user requests |
| OpenAI Model - Simple Agent(mini-model) | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backing simple task tool |  | Simple Task Agent Tool | ## Simple Task Flow<br>Handles quick, single-step user requests |
| Complex Task Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Tool for complex requests | OpenAI Model - Complex Agent | Execute Selected Agent | ## Complex Task Flow<br>Handles multi-step reasoning and analysis |
| OpenAI Model - Complex Agent | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backing complex task tool |  | Complex Task Agent Tool | ## Complex Task Flow<br>Handles multi-step reasoning and analysis |
| Send Fallback Alert | n8n-nodes-base.emailSend | Send human-review alert for low-confidence classification | Check Confidence Score |  | ## Fallback Handling<br>Send email alert when confidence is low |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node**
   - Type: `Webhook`
   - HTTP path: set a custom path, for example the generated UUID-like value used here
   - Response mode: `Last Node`
   - Leave other options default unless you need custom HTTP method or response handling

3. **Add a Set node named `Workflow Configuration`**
   - Type: `Set`
   - Add field `userRequest` as `String`
   - Add field `confidenceThreshold` as `Number`
   - Set `confidenceThreshold` to `0.7`
   - Enable `Include Other Input Fields`
   - Important: in a production-ready version, replace the placeholder with an expression from the webhook payload, such as a body field containing the user request

4. **Connect `Webhook` → `Workflow Configuration`**

5. **Add an AI Agent node named `Supervisor Agent`**
   - Type: `AI Agent` / LangChain Agent
   - Prompt type: `Define`
   - Text: `{{ $json.userRequest }}`
   - Enable structured output parser support
   - Use this system behavior:
     - analyze request complexity
     - return `simple` or `complex`
     - return confidence from 0 to 1
     - return short reasoning

6. **Add an OpenAI Chat Model node named `OpenAI Model - Supervisor`**
   - Type: OpenAI Chat Model
   - Model: `gpt-4.1-mini`
   - Attach OpenAI credentials

7. **Connect `OpenAI Model - Supervisor` to `Supervisor Agent`** using the AI language model connection.

8. **Add a Structured Output Parser node named `Routing Decision Parser`**
   - Type: Structured Output Parser
   - Schema type: manual
   - Use a schema with:
     - `complexity`: string enum `simple`, `complex`
     - `confidence`: number, min `0`, max `1`
     - `reasoning`: string
   - Mark all three fields as required

9. **Connect `Routing Decision Parser` to `Supervisor Agent`** using the AI output parser connection.

10. **Connect `Workflow Configuration` → `Supervisor Agent`**

11. **Add an IF node named `Check Confidence Score`**
   - Type: `IF`
   - Condition:
     - left value: `{{ $json.confidence }}`
     - operator: number `greater than or equal`
     - right value: `{{ $('Workflow Configuration').first().json.confidenceThreshold }}`
   - Keep validation loose if you want parity with the source workflow

12. **Connect `Supervisor Agent` → `Check Confidence Score`**

13. **Add an AI Agent node named `Execute Selected Agent`**
   - Type: `AI Agent` / LangChain Agent
   - Prompt type: `Define`
   - Text: `{{ $('Workflow Configuration').first().json.userRequest }}`
   - System prompt should instruct the agent to:
     - analyze the request
     - use the supervisor classification
     - call the simple tool for simple requests
     - call the complex tool for complex requests
     - return the selected tool result
   - Include in the system prompt:
     - `{{ $json.complexity }}`
     - `{{ $json.reasoning }}`

14. **Add an OpenAI Chat Model node named `OpenAI Model - Executor`**
   - Model: `gpt-4.1-mini`
   - Attach the same or another OpenAI credential

15. **Connect `OpenAI Model - Executor` to `Execute Selected Agent`** using the AI language model connection.

16. **Add an Agent Tool node named `Simple Task Agent Tool`**
   - Type: `Agent Tool`
   - Tool description: indicate that it handles straightforward tasks
   - Text/input: `{{ $fromAI("task", "The task to process") }}`
   - System message should emphasize:
     - concise answers
     - basic questions
     - single-step tasks
     - simple information retrieval

17. **Add an OpenAI Chat Model node named `OpenAI Model - Simple Agent(mini-model)`**
   - Model: `gpt-4.1-mini`
   - Attach OpenAI credentials

18. **Connect `OpenAI Model - Simple Agent(mini-model)` to `Simple Task Agent Tool`** via AI language model connection.

19. **Connect `Simple Task Agent Tool` to `Execute Selected Agent`** via AI tool connection.

20. **Add an Agent Tool node named `Complex Task Agent Tool`**
   - Type: `Agent Tool`
   - Tool description: indicate that it handles sophisticated tasks requiring analysis or reasoning
   - Text/input: `{{ $fromAI("task", "The task to process") }}`
   - System message should emphasize:
     - multi-step reasoning
     - detailed analysis
     - domain expertise
     - comprehensive responses

21. **Add an OpenAI Chat Model node named `OpenAI Model - Complex Agent`**
   - Model: `gpt-4.1-mini`
   - Attach OpenAI credentials

22. **Connect `OpenAI Model - Complex Agent` to `Complex Task Agent Tool`** via AI language model connection.

23. **Connect `Complex Task Agent Tool` to `Execute Selected Agent`** via AI tool connection.

24. **Connect the true output of `Check Confidence Score` → `Execute Selected Agent`**

25. **Add an Email Send node named `Send Fallback Alert`**
   - Type: `Send Email`
   - Configure SMTP or mail credentials supported by your n8n instance
   - To email: admin/reviewer address
   - From email: valid sender address
   - Subject: `Low Confidence Alert: Task Classification Uncertain`
   - Email format: `Text`
   - Body should include:
     - original request: `{{ $('Workflow Configuration').first().json.userRequest }}`
     - complexity: `{{ $json.complexity }}`
     - confidence: `{{ $json.confidence }}`
     - reasoning: `{{ $json.reasoning }}`
     - threshold: `{{ $('Workflow Configuration').first().json.confidenceThreshold }}`

26. **Connect the false output of `Check Confidence Score` → `Send Fallback Alert`**

27. **Configure credentials**
   - **OpenAI credentials:** required for
     - `OpenAI Model - Supervisor`
     - `OpenAI Model - Executor`
     - `OpenAI Model - Simple Agent(mini-model)`
     - `OpenAI Model - Complex Agent`
   - **Email credentials:** required for `Send Fallback Alert`

28. **Optional but recommended improvement**
   - Replace the static `userRequest` placeholder in `Workflow Configuration` with a webhook expression, for example from request body JSON.
   - Otherwise, the workflow will repeatedly process the placeholder text instead of actual requests.

29. **Test the workflow**
   - Send a webhook request with sample task text
   - Confirm:
     - the supervisor returns valid structured output
     - the IF condition branches correctly
     - the executor calls one of the two tools when confidence is high
     - email fallback works when confidence is low

30. **Activate the workflow**
   - Use the production webhook URL
   - Verify OpenAI model availability and email deliverability before live usage

### Sub-workflow setup
This workflow does **not** invoke any sub-workflow and does not contain any `Execute Workflow` node. No separate child workflow is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow receives a user request via webhook and analyzes it using a supervisor AI agent. The agent classifies the request as either simple or complex and assigns a confidence score. | Overall workflow note |
| If the confidence meets the threshold, the workflow routes the task to the appropriate agent (simple or complex) using an orchestrator agent. Each agent processes the request using an LLM and returns a response. | Overall workflow note |
| If the confidence is too low, the workflow sends an email alert for manual review instead of executing the task. | Overall workflow note |
| Add OpenAI credentials for all AI nodes. | Setup note |
| Configure the webhook endpoint. | Setup note |
| Set sender and recipient email in alert node. | Setup note |
| Adjust confidence threshold if needed. | Setup note |
| Customize system prompts for agents. | Setup note |