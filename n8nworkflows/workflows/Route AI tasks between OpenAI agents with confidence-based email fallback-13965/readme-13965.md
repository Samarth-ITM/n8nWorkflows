Route AI tasks between OpenAI agents with confidence-based email fallback

https://n8nworkflows.xyz/workflows/route-ai-tasks-between-openai-agents-with-confidence-based-email-fallback-13965


# Route AI tasks between OpenAI agents with confidence-based email fallback

## 1. Workflow Overview

This workflow receives an incoming request via a webhook, asks a supervisory AI agent to classify the request as **simple** or **complex**, checks how confident that classification is, and then either:

- routes the task to the appropriate AI execution path, or
- sends an email alert for human review if confidence is too low.

Its main use case is **controlled AI task routing**: using one model to classify request complexity, then using an orchestration agent to delegate execution to one of two specialized AI tools. It also introduces a **safety fallback** by notifying an administrator when the classification is uncertain.

### 1.1 Input Reception and Runtime Configuration
The workflow starts from a webhook and stores runtime values needed later, especially the user request and the confidence threshold used to decide whether automation is safe.

### 1.2 Supervisory Classification
A supervisor agent analyzes the request and produces structured JSON containing:
- `complexity`: `simple` or `complex`
- `confidence`: numeric score from 0 to 1
- `reasoning`: explanation for the decision

### 1.3 Confidence Gate
The workflow compares the supervisor’s confidence score to a configured threshold. High-confidence classifications proceed to automated execution; low-confidence ones trigger an email alert.

### 1.4 AI Orchestration and Specialized Execution
If confidence is high enough, a second AI agent acts as an orchestrator. It uses the supervisor’s classification and reasoning to decide whether to call:
- a simple task tool for lightweight requests, or
- a complex task tool for multi-step or deeper reasoning.

### 1.5 Human Fallback Notification
If the confidence score is below threshold, the workflow sends a plain-text email containing the request, classification, confidence, reasoning, and threshold, asking for manual review.

### 1.6 Webhook Response
When automated execution succeeds, the workflow responds to the original webhook request using a dedicated response node.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Runtime Configuration

**Overview:**  
This block accepts the incoming HTTP request and prepares core values used by later nodes. It currently injects placeholder configuration values rather than extracting them from the webhook payload.

**Nodes Involved:**  
- Webhook
- Workflow Configuration

### Node: Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for the workflow. Receives an HTTP request and starts execution.
- **Configuration choices:**  
  - Uses a generated path: `b62c065b-ab0a-41f5-81f8-c5207fbcd892`
  - `responseMode` is set to **responseNode**, meaning the HTTP response is not returned immediately by the webhook node itself; instead, the workflow must later use a **Respond to Webhook** node.
- **Key expressions or variables used:**  
  None configured directly in this node.
- **Input and output connections:**  
  - No input; this is the trigger.
  - Outputs to **Workflow Configuration**.
- **Version-specific requirements:**  
  Type version `2.1`.
- **Edge cases or potential failure types:**  
  - If no Respond to Webhook node executes, callers may experience timeout or incomplete responses.
  - If deployed behind a proxy or with incorrect webhook URL setup, requests may not reach n8n.
  - Authentication is not configured here, so the endpoint is effectively open unless protected externally.
- **Sub-workflow reference:**  
  None.

### Node: Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines reusable workflow values.
- **Configuration choices:**  
  - Sets `userRequest` to a placeholder string: `<__PLACEHOLDER_VALUE__User task request__>`
  - Sets `confidenceThreshold` to `0.7`
  - `includeOtherFields` is enabled, so any incoming webhook data is preserved alongside these assigned values.
- **Key expressions or variables used:**  
  No dynamic expressions in assignments; values are hardcoded placeholders.
- **Input and output connections:**  
  - Input from **Webhook**
  - Output to **Supervisor Agent**
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - As configured, this node does **not** map the actual webhook payload into `userRequest`; unless manually changed, the supervisor will always receive the placeholder text.
  - If downstream nodes assume real user input, results will be misleading.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Supervisory Classification

**Overview:**  
This block classifies the incoming request as simple or complex. It uses an OpenAI chat model plus a structured output parser to enforce a machine-readable routing decision.

**Nodes Involved:**  
- Supervisor Agent
- OpenAI Model - Supervisor
- Routing Decision Parser

### Node: Supervisor Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent that analyzes the request and produces a structured classification.
- **Configuration choices:**  
  - Prompt source: `={{ $json.userRequest }}`
  - Prompt type: `define`
  - System message instructs the model to:
    1. analyze request complexity
    2. classify as `simple` or `complex`
    3. assign a confidence score
    4. provide reasoning
  - `hasOutputParser` is enabled, so it must conform to the attached structured schema.
- **Key expressions or variables used:**  
  - `{{ $json.userRequest }}`
- **Input and output connections:**  
  - Main input from **Workflow Configuration**
  - AI language model input from **OpenAI Model - Supervisor**
  - AI output parser input from **Routing Decision Parser**
  - Main output to **Check Confidence Score**
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If the model returns invalid JSON or misses required fields, the parser may fail.
  - Ambiguous input may lead to inconsistent confidence values.
  - Missing OpenAI credentials will prevent execution.
- **Sub-workflow reference:**  
  None.

### Node: OpenAI Model - Supervisor
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Language model backend used by the Supervisor Agent.
- **Configuration choices:**  
  - Model selected: `gpt-4.1-mini`
  - No additional model options are defined.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model output to **Supervisor Agent**
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Invalid or missing OpenAI credentials
  - Model availability or rate limiting
  - Token/context issues if input becomes large
- **Sub-workflow reference:**  
  None.

### Node: Routing Decision Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a JSON schema on the supervisor’s response.
- **Configuration choices:**  
  Manual schema with:
  - `complexity`: string enum `simple` or `complex`
  - `confidence`: number between 0 and 1
  - `reasoning`: string
  All three fields are required.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI output parser output to **Supervisor Agent**
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Model outputs incompatible structure
  - Numeric confidence outside `[0,1]`
  - Empty or malformed JSON
- **Sub-workflow reference:**  
  None.

---

## 2.3 Confidence Gate

**Overview:**  
This block decides whether the automated route is trusted enough to proceed. It compares the supervisor’s confidence score to the configured threshold.

**Nodes Involved:**  
- Check Confidence Score

### Node: Check Confidence Score
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branch controlling automated execution versus fallback.
- **Configuration choices:**  
  - Single condition:
    - left value: `={{ $json.confidence }}`
    - operator: number `>=`
    - right value: `={{ $('Workflow Configuration').first().json.confidenceThreshold }}`
  - Strict type validation is enabled.
- **Key expressions or variables used:**  
  - `{{ $json.confidence }}`
  - `{{ $('Workflow Configuration').first().json.confidenceThreshold }}`
- **Input and output connections:**  
  - Input from **Supervisor Agent**
  - True output to **Execute Selected Agent**
  - False output to **Send Fallback Alert**
- **Version-specific requirements:**  
  Type version `2.3`.
- **Edge cases or potential failure types:**  
  - If `confidence` is missing or not numeric, strict validation may cause branch errors.
  - If the threshold is changed to a non-number, comparison may fail.
- **Sub-workflow reference:**  
  None.

---

## 2.4 AI Orchestration and Specialized Execution

**Overview:**  
This block performs the actual automated handling of the request. An orchestrator agent receives the original task plus the supervisor’s classification and can call one of two specialized tool-agents.

**Nodes Involved:**  
- Execute Selected Agent
- OpenAI Model - Executor
- Simple Task Agent Tool(5 MINI)
- OpenAI Model - Simple Agent
- Complex Task Agent Tool (5.4)
- OpenAI Model - Complex Agent GPT 5.3

### Node: Execute Selected Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Orchestrator agent that chooses which AI tool to call.
- **Configuration choices:**  
  - Text input: `={{ $('Workflow Configuration').first().json.userRequest }}`
  - Prompt type: `define`
  - System message instructs it to:
    1. analyze the user request
    2. determine which specialized agent tool to call
    3. use Simple Task Agent Tool for simple requests
    4. use Complex Task Agent Tool for complex requests
    5. return the selected agent’s result
  - The system message includes:
    - `Task complexity classification: {{ $json.complexity }}`
    - `Reasoning: {{ $json.reasoning }}`
- **Key expressions or variables used:**  
  - `{{ $('Workflow Configuration').first().json.userRequest }}`
  - `{{ $json.complexity }}`
  - `{{ $json.reasoning }}`
- **Input and output connections:**  
  - Main input from **Check Confidence Score**
  - AI language model input from **OpenAI Model - Executor**
  - AI tool inputs from:
    - **Simple Task Agent Tool(5 MINI)**
    - **Complex Task Agent Tool (5.4)**
  - Main output to **Respond to Webhook**
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If the model ignores the classification and calls the wrong tool, routing quality degrades.
  - If no tool is called and the agent responds directly, behavior may differ from expected design.
  - Missing OpenAI credentials or tool connection problems will stop execution.
- **Sub-workflow reference:**  
  None.

### Node: OpenAI Model - Executor
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Model used by the orchestrator agent.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
  - No extra options set.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model output to **Execute Selected Agent**
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Same OpenAI issues as above: auth, rate limits, model availability
- **Sub-workflow reference:**  
  None.

### Node: Simple Task Agent Tool(5 MINI)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized callable AI tool for basic requests.
- **Configuration choices:**  
  - Tool input text: `={{ $fromAI("task", "The task to process") }}`
  - System message positions this tool as a simple-task specialist for:
    - basic questions
    - simple information
    - single-step operations
    - quick concise answers
  - Tool description: straightforward tasks and simple information retrieval
- **Key expressions or variables used:**  
  - `{{ $fromAI("task", "The task to process") }}`
- **Input and output connections:**  
  - AI language model input from **OpenAI Model - Simple Agent**
  - AI tool output to **Execute Selected Agent**
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - If the orchestrator passes poor task content, the answer quality drops.
  - The tool depends on the model node being correctly attached.
- **Sub-workflow reference:**  
  None.

### Node: OpenAI Model - Simple Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Model powering the simple-task tool.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model output to **Simple Task Agent Tool(5 MINI)**
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - OpenAI auth/rate limit/model issues
- **Sub-workflow reference:**  
  None.

### Node: Complex Task Agent Tool (5.4)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized callable AI tool for advanced requests.
- **Configuration choices:**  
  - Tool input text: `={{ $fromAI("task", "The task to process") }}`
  - System message defines it as a complex-task specialist for:
    - multi-step reasoning
    - in-depth analysis
    - domain expertise
    - creative problem-solving
    - detailed explanations
  - Tool description emphasizes sophisticated requests requiring analysis or expertise.
- **Key expressions or variables used:**  
  - `{{ $fromAI("task", "The task to process") }}`
- **Input and output connections:**  
  - AI language model input from **OpenAI Model - Complex Agent GPT 5.3**
  - AI tool output to **Execute Selected Agent**
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Same tool invocation risks as the simple tool
  - Potentially larger responses, which may affect latency or token usage
- **Sub-workflow reference:**  
  None.

### Node: OpenAI Model - Complex Agent GPT 5.3
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Model powering the complex-task tool.
- **Configuration choices:**  
  - Despite the node name, the configured model value is `gpt-4.1-mini`
  - No advanced options are set
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI language model output to **Complex Task Agent Tool (5.4)**
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Node name and actual model selection are inconsistent, which can confuse maintainers
  - Same OpenAI auth/rate-limit/model issues
- **Sub-workflow reference:**  
  None.

---

## 2.5 Human Fallback Notification

**Overview:**  
This branch handles uncertain classifications. Instead of executing the task automatically, it sends an email to an administrator for manual review.

**Nodes Involved:**  
- Send Fallback Alert

### Node: Send Fallback Alert
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends a plain-text email alert.
- **Configuration choices:**  
  - Subject: `Low Confidence Alert: Task Classification Uncertain`
  - Plain text body includes:
    - original user request
    - classification
    - confidence score
    - reasoning
    - configured threshold
  - `toEmail` and `fromEmail` are placeholders and must be replaced
  - `emailFormat` is `text`
- **Key expressions or variables used:**  
  - `{{ $('Workflow Configuration').first().json.userRequest }}`
  - `{{ $json.complexity }}`
  - `{{ $json.confidence }}`
  - `{{ $json.reasoning }}`
  - `{{ $('Workflow Configuration').first().json.confidenceThreshold }}`
- **Input and output connections:**  
  - Input from the false branch of **Check Confidence Score**
  - No output connected
- **Version-specific requirements:**  
  Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Missing email credentials or SMTP misconfiguration
  - Placeholder sender/recipient addresses will cause delivery failure
  - This branch does **not** connect to **Respond to Webhook**, so the webhook caller may not receive a response in fallback scenarios
- **Sub-workflow reference:**  
  None.

---

## 2.6 Webhook Response

**Overview:**  
This block returns the final automated result to the original HTTP caller. It only executes on the successful high-confidence automation branch.

**Nodes Involved:**  
- Respond to Webhook

### Node: Respond to Webhook
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the HTTP response for workflows using webhook `responseMode: responseNode`.
- **Configuration choices:**  
  - No custom response options configured.
  - It will return the incoming item data from **Execute Selected Agent** by default.
- **Key expressions or variables used:**  
  None explicitly configured.
- **Input and output connections:**  
  - Input from **Execute Selected Agent**
- **Version-specific requirements:**  
  Type version `1.5`.
- **Edge cases or potential failure types:**  
  - If upstream data is overly large or malformed, HTTP response payload may be undesirable
  - Since no fallback branch connects here, low-confidence executions may leave the request unanswered
- **Sub-workflow reference:**  
  None.

---

## 2.7 Sticky Notes and In-Canvas Documentation

**Overview:**  
These nodes are non-executable documentation elements placed on the canvas. They explain the intent of major parts of the workflow.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node: Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation for task classification area.
- **Configuration choices:**  
  Content explains that the Supervisor Agent analyzes the request and determines if it is simple or complex.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

### Node: Sticky Note1
- Same node type and purpose as above.  
  Documents the low-confidence alert behavior.

### Node: Sticky Note2
- Same node type and purpose as above.  
  Documents orchestration behavior between the supervisor result and the specialized tools.

### Node: Sticky Note3
- Same node type and purpose as above.  
  Documents the distinction between simple and complex agents.

### Node: Sticky Note4
- Same node type and purpose as above.  
  States that the workflow starts from a webhook trigger.

### Node: Sticky Note5
- Same node type and purpose as above.  
  Labels the confidence-check area.

### Node: Sticky Note6
- Same node type and purpose as above.  
  Provides a broad description of the workflow’s routing logic and human fallback.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | HTTP trigger entry point |  | Workflow Configuration | ## Workflow Start  A WEBHOOK trigger starts the execution  |
| Workflow Configuration | n8n-nodes-base.set | Stores request and threshold values | Webhook | Supervisor Agent | ## AI Task Routing with Supervisor Agent  This workflow intelligently analyzes incoming user requests and routes them to the most appropriate AI agent based on task complexity. When a request is received through the webhook, it is first stored in the configuration step along with a confidence threshold.  A Supervisor Agent then evaluates the request and classifies it as either simple or complex, returning a confidence score and reasoning for its decision. The workflow checks whether this confidence score meets the defined threshold.  If the confidence is sufficient, an orchestrator agent delegates the task to a specialized AI tool. Simple requests are handled by a Simple Task Agent, while more advanced requests are processed by a Complex Task Agent capable of deeper reasoning.  If the confidence score is too low, the workflow sends an email alert for human review, ensuring reliable and controlled AI automation. |
| Supervisor Agent | @n8n/n8n-nodes-langchain.agent | Classifies task complexity with confidence and reasoning | Workflow Configuration; OpenAI Model - Supervisor; Routing Decision Parser | Check Confidence Score | ## Task Classification  The Supervisor Agent analyzes the user request and determines whether the task is simple or complex. |
| Routing Decision Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON classification output |  | Supervisor Agent | ## Task Classification  The Supervisor Agent analyzes the user request and determines whether the task is simple or complex. |
| OpenAI Model - Supervisor | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for supervisor classification |  | Supervisor Agent | ## Task Classification  The Supervisor Agent analyzes the user request and determines whether the task is simple or complex. |
| Check Confidence Score | n8n-nodes-base.if | Routes high-confidence vs low-confidence outcomes | Supervisor Agent | Execute Selected Agent; Send Fallback Alert | ## Check Confidence Score |
| Execute Selected Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates which specialized AI tool should handle the request | Check Confidence Score; OpenAI Model - Executor; Simple Task Agent Tool(5 MINI); Complex Task Agent Tool (5.4) | Respond to Webhook | ## Agent Orchestration  An orchestrator AI agent decides which specialized tool to call based on the supervisor’s classification. The request is  either the Simple Task Agent or the Complex Task Agent |
| OpenAI Model - Executor | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for orchestration agent |  | Execute Selected Agent | ## Agent Orchestration  An orchestrator AI agent decides which specialized tool to call based on the supervisor’s classification. The request is  either the Simple Task Agent or the Complex Task Agent |
| Send Fallback Alert | n8n-nodes-base.emailSend | Sends low-confidence human review email | Check Confidence Score |  | ##  Alert  If the confidence is below the configured threshold, an email alert is sent to an administrator requesting manual review to prevent incorrect automated responses. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Returns final HTTP response | Execute Selected Agent |  |  |
| OpenAI Model - Complex Agent GPT 5.3 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for complex-task tool |  | Complex Task Agent Tool (5.4) | ## Two specialized agents handle task execution:  Simple Task Agent for straightforward questions and quick responses.  Complex Task Agent for multi-step reasoning, deeper analysis, and detailed explanations.  This separation improves response quality and efficiency. |
| Complex Task Agent Tool (5.4) | @n8n/n8n-nodes-langchain.agentTool | Tool callable by orchestrator for complex tasks | OpenAI Model - Complex Agent GPT 5.3 | Execute Selected Agent | ## Two specialized agents handle task execution:  Simple Task Agent for straightforward questions and quick responses.  Complex Task Agent for multi-step reasoning, deeper analysis, and detailed explanations.  This separation improves response quality and efficiency. |
| Simple Task Agent Tool(5 MINI) | @n8n/n8n-nodes-langchain.agentTool | Tool callable by orchestrator for simple tasks | OpenAI Model - Simple Agent | Execute Selected Agent | ## Two specialized agents handle task execution:  Simple Task Agent for straightforward questions and quick responses.  Complex Task Agent for multi-step reasoning, deeper analysis, and detailed explanations.  This separation improves response quality and efficiency. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| OpenAI Model - Simple Agent | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for simple-task tool |  | Simple Task Agent Tool(5 MINI) | ## Two specialized agents handle task execution:  Simple Task Agent for straightforward questions and quick responses.  Complex Task Agent for multi-step reasoning, deeper analysis, and detailed explanations.  This separation improves response quality and efficiency. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node**
   - Type: **Webhook**
   - Set the HTTP path to any unique value, for example the generated UUID-style path used in the source workflow.
   - Set **Response Mode** to **Using Respond to Webhook Node** / `responseNode`.

3. **Add a Set node named “Workflow Configuration”**
   - Type: **Set**
   - Add field `userRequest` as **string**
   - Add field `confidenceThreshold` as **number**
   - Set `confidenceThreshold` to `0.7`
   - In the provided workflow, `userRequest` is a placeholder value. For a real implementation, replace it with something from the webhook, such as:
     - `{{$json.body.userRequest}}` or
     - `{{$json.query.userRequest}}`
     depending on how the webhook caller sends data.
   - Enable **Include Other Input Fields**.

4. **Connect Webhook → Workflow Configuration**

5. **Add an AI Agent node named “Supervisor Agent”**
   - Type: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Set **Prompt Type** to **Define below**
   - Set **Text** to:
     - `={{ $json.userRequest }}`
   - Enable structured output parsing.
   - Set system message to instruct the model to:
     - classify as `simple` or `complex`
     - produce a confidence score between `0.0` and `1.0`
     - provide reasoning

6. **Add an OpenAI Chat Model node named “OpenAI Model - Supervisor”**
   - Type: **OpenAI Chat Model**
   - Choose model `gpt-4.1-mini`
   - Attach valid **OpenAI credentials**

7. **Connect OpenAI Model - Supervisor → Supervisor Agent** using the **AI Language Model** connection.

8. **Add a Structured Output Parser node named “Routing Decision Parser”**
   - Type: **Structured Output Parser**
   - Use manual JSON schema:
     - object with required fields:
       - `complexity` as string enum `simple` / `complex`
       - `confidence` as number 0 to 1
       - `reasoning` as string

9. **Connect Routing Decision Parser → Supervisor Agent** using the **AI Output Parser** connection.

10. **Connect Workflow Configuration → Supervisor Agent** via the main connection.

11. **Add an IF node named “Check Confidence Score”**
   - Type: **IF**
   - Configure one numeric condition:
     - left: `={{ $json.confidence }}`
     - operator: **greater than or equal**
     - right: `={{ $('Workflow Configuration').first().json.confidenceThreshold }}`
   - Keep strict type validation enabled.

12. **Connect Supervisor Agent → Check Confidence Score**

13. **Add an AI Agent node named “Execute Selected Agent”**
   - Type: **AI Agent**
   - Set **Prompt Type** to **Define below**
   - Set **Text** to:
     - `={{ $('Workflow Configuration').first().json.userRequest }}`
   - Set system message so the agent:
     - analyzes the request
     - chooses the specialized tool based on supervisor classification
     - calls simple tool for simple tasks
     - calls complex tool for complex tasks
     - returns the selected agent’s output
   - Include these dynamic references in the system message:
     - `Task complexity classification: {{ $json.complexity }}`
     - `Reasoning: {{ $json.reasoning }}`

14. **Add an OpenAI Chat Model node named “OpenAI Model - Executor”**
   - Type: **OpenAI Chat Model**
   - Model: `gpt-4.1-mini`
   - Use valid OpenAI credentials

15. **Connect OpenAI Model - Executor → Execute Selected Agent** via **AI Language Model**.

16. **Add a Tool Agent node named “Simple Task Agent Tool(5 MINI)”**
   - Type: **Agent Tool**
   - Set tool input text to:
     - `={{ $fromAI("task", "The task to process") }}`
   - Set the system message to define it as a simple-task specialist:
     - basic questions
     - quick information
     - single-step operations
     - concise responses
   - Add tool description similar to:
     - “Handles straightforward tasks like basic questions and simple information retrieval”

17. **Add an OpenAI Chat Model node named “OpenAI Model - Simple Agent”**
   - Type: **OpenAI Chat Model**
   - Model: `gpt-4.1-mini`
   - Use valid OpenAI credentials

18. **Connect OpenAI Model - Simple Agent → Simple Task Agent Tool(5 MINI)** via **AI Language Model**.

19. **Connect Simple Task Agent Tool(5 MINI) → Execute Selected Agent** via **AI Tool**.

20. **Add a Tool Agent node named “Complex Task Agent Tool (5.4)”**
   - Type: **Agent Tool**
   - Set tool input text to:
     - `={{ $fromAI("task", "The task to process") }}`
   - Set system message to define it as a complex-task specialist:
     - multi-step reasoning
     - deep analysis
     - domain expertise
     - creative problem-solving
     - detailed explanations
   - Add tool description similar to:
     - “Handles complex tasks requiring multi-step reasoning, analysis, or domain expertise”

21. **Add an OpenAI Chat Model node named “OpenAI Model - Complex Agent GPT 5.3”**
   - Type: **OpenAI Chat Model**
   - In the original workflow, the node name suggests a different model, but the actual configured model is `gpt-4.1-mini`
   - You may keep the original naming for parity or rename it for clarity
   - Use valid OpenAI credentials

22. **Connect OpenAI Model - Complex Agent GPT 5.3 → Complex Task Agent Tool (5.4)** via **AI Language Model**.

23. **Connect Complex Task Agent Tool (5.4) → Execute Selected Agent** via **AI Tool**.

24. **Connect the true output of Check Confidence Score → Execute Selected Agent**

25. **Add an Email Send node named “Send Fallback Alert”**
   - Type: **Send Email**
   - Configure SMTP or the email transport supported in your n8n setup
   - Set **To Email** to the admin recipient
   - Set **From Email** to a valid sender
   - Set **Subject** to:
     - `Low Confidence Alert: Task Classification Uncertain`
   - Set email format to **text**
   - Use a body containing:
     - original request
     - classification
     - confidence score
     - reasoning
     - configured threshold
   - Suggested expressions:
     - `{{ $('Workflow Configuration').first().json.userRequest }}`
     - `{{ $json.complexity }}`
     - `{{ $json.confidence }}`
     - `{{ $json.reasoning }}`
     - `{{ $('Workflow Configuration').first().json.confidenceThreshold }}`

26. **Connect the false output of Check Confidence Score → Send Fallback Alert**

27. **Add a Respond to Webhook node named “Respond to Webhook”**
   - Type: **Respond to Webhook**
   - Default response settings are acceptable if you want to return the agent result as-is.

28. **Connect Execute Selected Agent → Respond to Webhook**

29. **Optionally add sticky notes** to document:
   - workflow start
   - task classification
   - confidence check
   - agent orchestration
   - specialized agents
   - alert behavior
   - high-level workflow purpose

30. **Configure credentials**
   - **OpenAI credentials** for:
     - OpenAI Model - Supervisor
     - OpenAI Model - Executor
     - OpenAI Model - Simple Agent
     - OpenAI Model - Complex Agent GPT 5.3
   - **Email/SMTP credentials** for:
     - Send Fallback Alert

31. **Test the workflow**
   - Send a webhook request containing a real `userRequest`
   - Confirm the supervisor returns valid structured JSON
   - Confirm high-confidence requests reach the orchestrator and return via Respond to Webhook
   - Confirm low-confidence requests generate email alerts

32. **Recommended fixes before production**
   - Replace the placeholder `userRequest` assignment with actual webhook input mapping
   - Add a webhook response for the low-confidence branch as well, otherwise callers may wait indefinitely
   - Consider adding webhook authentication
   - Rename the complex model node if you want its label to match the actual configured model

### Sub-workflow setup
There are **no sub-workflows** and no **Execute Workflow** nodes in this workflow. All logic is implemented within a single workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Task Routing with Supervisor Agent: incoming requests are analyzed, classified by complexity, then either routed to a specialized AI agent or escalated for human review if confidence is too low. | Canvas documentation |
| Workflow Start: a webhook trigger starts the execution. | Canvas documentation |
| Task Classification: the Supervisor Agent determines whether the task is simple or complex. | Canvas documentation |
| Check Confidence Score | Canvas documentation |
| Agent Orchestration: an orchestrator AI agent decides which specialized tool to call based on the supervisor’s classification. The request is either the Simple Task Agent or the Complex Task Agent. | Canvas documentation |
| Two specialized agents handle task execution: Simple Task Agent for straightforward questions and quick responses; Complex Task Agent for multi-step reasoning, deeper analysis, and detailed explanations. This separation improves response quality and efficiency. | Canvas documentation |
| Alert: if confidence is below the configured threshold, an email alert is sent to an administrator requesting manual review to prevent incorrect automated responses. | Canvas documentation |

## Additional implementation cautions
- The current workflow responds to the webhook only on the **high-confidence path**.
- The current `Workflow Configuration` node uses a **placeholder request** instead of actual webhook data.
- The node named **OpenAI Model - Complex Agent GPT 5.3** is actually configured to use **`gpt-4.1-mini`**.