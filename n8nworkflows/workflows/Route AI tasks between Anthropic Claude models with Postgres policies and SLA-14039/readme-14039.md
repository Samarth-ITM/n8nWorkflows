Route AI tasks between Anthropic Claude models with Postgres policies and SLA

https://n8nworkflows.xyz/workflows/route-ai-tasks-between-anthropic-claude-models-with-postgres-policies-and-sla-14039


# Route AI tasks between Anthropic Claude models with Postgres policies and SLA

# 1. Workflow Overview

This workflow is a policy-driven LLM orchestrator built in n8n. It receives tasks through an HTTP webhook, classifies each task, looks up routing policies in Postgres, decides whether to send the task to a smaller or larger Anthropic Claude model, records telemetry, and returns the AI result to the caller. In parallel, a second entry point runs weekly to analyze historical telemetry and automatically update routing policies.

Typical use cases:
- Cost-aware AI routing between cheaper and stronger models
- SLA-aware model selection based on latency and priority
- Adaptive AI orchestration with database-backed policies
- Continuous optimization of model routing from production telemetry

## 1.1 Input Reception and Base Configuration
The workflow starts from a webhook that accepts task requests. A Set node injects default orchestration constraints such as latency, token, cost, and retry limits.

## 1.2 Task Classification
A LangChain agent uses an Anthropic model plus a structured output parser to classify the incoming task into one of four task types and assign a confidence score.

## 1.3 Policy Retrieval and Decision Engine
The workflow queries Postgres for policy rules matching the classified task type and priority, then combines policy data with workflow defaults to produce a final routing decision.

## 1.4 Model Routing and Execution
A Switch node routes work to either a “small” or “large” Anthropic execution path. Each path uses a dedicated LangChain agent backed by a different Claude model.

## 1.5 Telemetry and API Response
After model execution, the workflow estimates latency, token usage, cost, success state, and model metadata, stores the record in Postgres, and returns the result to the webhook caller.

## 1.6 Weekly Self-Tuning Optimization
A schedule trigger runs weekly, retrieves telemetry from the last 7 days, computes performance trends, generates routing recommendations, and updates the policy table in Postgres.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Base Configuration

**Overview:**  
This block exposes the workflow as an HTTP endpoint and establishes default operating limits used later by the policy engine. It is the main online entry point for task execution.

**Nodes Involved:**  
- Task Input Webhook  
- Workflow Configuration

### Task Input Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; receives external POST requests.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `llm-orchestrator`
  - Response mode: `lastNode`, meaning the workflow waits until the final responder completes.
- **Key expressions or variables used:** None directly in node config.
- **Input and output connections:**  
  - Entry point node
  - Outputs to `Workflow Configuration`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**  
  - Invalid payload structure, especially if `task` is missing
  - Long-running requests may hit webhook/client timeout before final response
  - If `Respond to Webhook` is misconfigured, callers may receive no proper payload
- **Sub-workflow reference:** None

### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`; injects orchestration defaults into the execution data.
- **Configuration choices:**  
  - Adds:
    - `maxLatencyMs = 30000`
    - `costCeiling = 0.5`
    - `timeoutMs = 25000`
    - `maxTokens = 4000`
    - `maxRetries = 2`
  - `includeOtherFields = true`, so incoming request fields remain available.
- **Key expressions or variables used:** Static values only.
- **Input and output connections:**  
  - Input from `Task Input Webhook`
  - Output to `Task Classifier Agent`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**  
  - Few direct failures; main risk is later logic assuming these values exist and are numeric
- **Sub-workflow reference:** None

---

## 2.2 Task Classification

**Overview:**  
This block uses an Anthropic chat model to classify the incoming task and produce structured metadata. The classification result drives policy lookup and downstream routing.

**Nodes Involved:**  
- Task Classifier Agent  
- Anthropic Chat Model - Classifier  
- Classification Output Parser

### Task Classifier Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates prompt execution with a connected language model and structured parser.
- **Configuration choices:**  
  - Prompt source: `={{ $json.task }}`
  - Prompt type: defined directly in node
  - Uses a system message instructing the model to classify into:
    - extraction
    - classification
    - reasoning
    - generation
  - Requires confidence score `0–100`
  - `hasOutputParser = true`
- **Key expressions or variables used:**  
  - `{{ $json.task }}`
- **Input and output connections:**  
  - Input from `Workflow Configuration`
  - AI language model input from `Anthropic Chat Model - Classifier`
  - AI output parser input from `Classification Output Parser`
  - Main output to `Load Policy Rules`
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**  
  - Missing `task` field causes poor or empty classification
  - Model may output invalid schema if parser enforcement fails
  - Priority is expected in parser output description, but the system prompt does not explicitly instruct how to derive it; it likely depends on input metadata or model inference
- **Sub-workflow reference:** None

### Anthropic Chat Model - Classifier
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; Anthropic Claude model used for classification.
- **Configuration choices:**  
  - Model: `claude-3-5-haiku-20241022`
  - No extra model options configured
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connected to `Task Classifier Agent` via `ai_languageModel`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**  
  - Anthropic credential/authentication failure
  - Rate limits
  - Model deprecation if this exact ID changes in Anthropic/n8n
- **Sub-workflow reference:** None

### Classification Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates/structures classifier output.
- **Configuration choices:**  
  - Manual JSON schema with fields:
    - `taskType` as string
    - `confidence` as number
    - `priority` as string
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connected to `Task Classifier Agent` via `ai_outputParser`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**  
  - Parser failure if model returns non-conforming output
  - `priority` may be missing or hallucinated if not passed clearly from input
- **Sub-workflow reference:** None

---

## 2.3 Policy Retrieval and Decision Engine

**Overview:**  
This block looks up database policies matching the classification result and merges them with workflow defaults to determine the final execution strategy. It is the core decision layer of the orchestration logic.

**Nodes Involved:**  
- Load Policy Rules  
- Policy Engine Decision

### Load Policy Rules
- **Type and technical role:** `n8n-nodes-base.postgres`; queries Postgres for matching policy rules.
- **Configuration choices:**  
  - Operation: `executeQuery`
  - SQL:
    - `SELECT * FROM policy_rules WHERE task_type = $1 AND priority = $2`
  - Query replacements:
    - `={{ $json.taskType }}`
    - `={{ $json.priority }}`
- **Key expressions or variables used:**  
  - `$json.taskType`
  - `$json.priority`
- **Input and output connections:**  
  - Input from `Task Classifier Agent`
  - Output to `Policy Engine Decision`
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**  
  - No rows returned for a task type/priority pair
  - Database connectivity or auth failure
  - If priority is missing from classification, query likely returns no match
  - Multiple rows may create multiple downstream executions unless uniqueness is enforced in DB
- **Sub-workflow reference:** None

### Policy Engine Decision
- **Type and technical role:** `n8n-nodes-base.code`; combines classification and policy data into a routing decision object.
- **Configuration choices:**  
  - Reads config from `Workflow Configuration`
  - Reads classification from `$input.first().json`
  - Reads current policy row from `$input.item.json`
  - Produces:
    - `modelSize`
    - `allowTools`
    - `retryStrategy`
    - `timeoutMs`
    - `maxTokens`
    - `costBudget`
    - `taskType`
    - `confidence`
    - `priority`
  - Applies overrides:
    - Low-confidence reasoning tasks force `large`
    - `critical` priority forces `large`
- **Key expressions or variables used:**  
  - `$('Workflow Configuration').first().json`
  - `$input.first().json`
  - `$input.item.json`
- **Input and output connections:**  
  - Input from `Load Policy Rules`
  - Output to `Route by Model Selection`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - This code likely has a data-shape risk:
    - both `classification` and `policy` are read from the current input context, but only `Load Policy Rules` is connected directly
    - unless the Postgres result still contains classification fields or item linking is preserved as expected, `classification.taskType`, `classification.confidence`, and `classification.priority` may be undefined
  - If no policy row exists, the node may never run
  - If `policy.max_latency_ms`, `policy.max_tokens`, or `policy.cost_limit` are null, defaults are used
- **Sub-workflow reference:** None

**Important implementation note:**  
For robust behavior, many builders would explicitly merge the classifier output with the DB result before the Code node, or reference the classifier directly with `$('Task Classifier Agent').first().json`.

---

## 2.4 Model Routing and Execution

**Overview:**  
This block routes the task to the appropriate execution model based on the policy decision. Lightweight tasks go to a smaller Claude model, while critical or complex tasks go to a larger one.

**Nodes Involved:**  
- Route by Model Selection  
- Small Model Execution  
- Anthropic Chat Model - Small  
- Large Model Execution  
- Anthropic Chat Model - Large

### Route by Model Selection
- **Type and technical role:** `n8n-nodes-base.switch`; branches execution by `modelSize`.
- **Configuration choices:**  
  - Output `small` if `={{ $json.modelSize }}` equals `small`
  - Output `large` if `={{ $json.modelSize }}` equals `large`
  - Fallback output enabled and renamed to `Fallback`
- **Key expressions or variables used:**  
  - `$json.modelSize`
- **Input and output connections:**  
  - Input from `Policy Engine Decision`
  - Output 0 to `Small Model Execution`
  - Output 1 to `Large Model Execution`
  - Fallback output is configured but not connected
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**  
  - Any value other than `small` or `large` is dropped into fallback and effectively ends unless connected
  - If `modelSize` is missing, same issue
- **Sub-workflow reference:** None

### Small Model Execution
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; executes lightweight tasks with a faster model.
- **Configuration choices:**  
  - Input text: `={{ $json.task }}`
  - System message emphasizes speed, efficiency, and concise responses
  - Mentions:
    - `{{ $json.taskType }}`
    - `{{ $json.priority }}`
  - No output parser attached
- **Key expressions or variables used:**  
  - `$json.task`
  - `$json.taskType`
  - `$json.priority`
- **Input and output connections:**  
  - Input from `Route by Model Selection`
  - AI language model input from `Anthropic Chat Model - Small`
  - Main output to `Prepare Telemetry Data`
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**  
  - The task text may be unavailable if earlier nodes did not preserve original input fields
  - No direct enforcement of `maxTokens`, `timeoutMs`, or `costBudget` in the model node config despite being computed upstream
  - Anthropic execution failures, timeouts, rate limits
- **Sub-workflow reference:** None

### Anthropic Chat Model - Small
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; backing model for small-route execution.
- **Configuration choices:**  
  - Model: `claude-3-5-haiku-20241022`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connected to `Small Model Execution`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**  
  - Same Anthropic auth/rate-limit/deprecation concerns as other model nodes
- **Sub-workflow reference:** None

### Large Model Execution
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; executes complex or high-priority tasks with a stronger model.
- **Configuration choices:**  
  - Input text: `={{ $json.task }}`
  - System message emphasizes complex reasoning and detailed analysis
  - Mentions:
    - `{{ $json.taskType }}`
    - `{{ $json.priority }}`
- **Key expressions or variables used:**  
  - `$json.task`
  - `$json.taskType`
  - `$json.priority`
- **Input and output connections:**  
  - Input from `Route by Model Selection`
  - AI language model input from `Anthropic Chat Model - Large`
  - Main output to `Prepare Telemetry Data`
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**  
  - Same data propagation risk as small-model path
  - No direct configuration enforcing policy-derived timeout/token budget
- **Sub-workflow reference:** None

### Anthropic Chat Model - Large
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; backing model for large-route execution.
- **Configuration choices:**  
  - Model: `claude-3-5-sonnet-20241022`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connected to `Large Model Execution`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**  
  - Anthropic auth failure
  - Higher cost and possibly higher latency than small model
  - Model ID future compatibility risk
- **Sub-workflow reference:** None

---

## 2.5 Telemetry and API Response

**Overview:**  
This block captures execution metadata, stores it in Postgres, and returns a JSON response to the original HTTP caller. It closes the main request path.

**Nodes Involved:**  
- Prepare Telemetry Data  
- Store Telemetry  
- Return Response

### Prepare Telemetry Data
- **Type and technical role:** `n8n-nodes-base.set`; enriches model output with telemetry fields.
- **Configuration choices:**  
  - Adds:
    - `executionLatencyMs = {{ $now.diff($('Task Classifier Agent').first().json.startTime) }}`
    - `modelUsed = {{ $json.modelSize || 'unknown' }}`
    - `tokensUsed = {{ $json.usage?.total_tokens || 0 }}`
    - `estimatedCost = {{ ($json.usage?.total_tokens || 0) * 0.00001 }}`
    - `success = true`
    - `failureType = "null"`
    - `timestamp = {{ $now.toISO() }}`
  - `includeOtherFields = true`
- **Key expressions or variables used:**  
  - `$now.diff(...)`
  - `$('Task Classifier Agent').first().json.startTime`
  - `$json.modelSize`
  - `$json.usage?.total_tokens`
  - `$now.toISO()`
- **Input and output connections:**  
  - Inputs from `Small Model Execution` and `Large Model Execution`
  - Output to `Store Telemetry`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**  
  - `startTime` is not clearly produced by `Task Classifier Agent`; latency expression may fail or return invalid data
  - `usage.total_tokens` may not exist depending on agent output shape
  - `modelSize` may not be preserved through the agent result, resulting in `unknown`
  - `failureType` is set as the string `"null"` rather than real null
- **Sub-workflow reference:** None

### Store Telemetry
- **Type and technical role:** `n8n-nodes-base.postgres`; inserts telemetry into the `public.telemetry` table.
- **Configuration choices:**  
  - Table: `public.telemetry`
  - Mapping mode: manual field mapping
  - Fields written:
    - `success`
    - `priority`
    - `task_type`
    - `timestamp`
    - `latency_ms`
    - `model_used`
    - `tokens_used`
    - `failure_type`
    - `estimated_cost`
- **Key expressions or variables used:**  
  - Uses fields like:
    - `$json.success`
    - `$json.priority`
    - `$json.task_type`
    - `$json.latency_ms`
    - `$json.model_used`
  - Note: these names differ from fields created in `Prepare Telemetry Data` (`executionLatencyMs`, `modelUsed`, `tokensUsed`, `estimatedCost`, `failureType`)
- **Input and output connections:**  
  - Input from `Prepare Telemetry Data`
  - Output to `Return Response`
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**  
  - Strong field-name mismatch risk:
    - upstream creates camelCase fields
    - Postgres mapping expects snake_case fields
  - `task_type` may not exist; upstream appears to use `taskType`
  - `latency_ms`, `model_used`, `tokens_used`, `failure_type`, `estimated_cost` may be null unless renamed before insert
  - DB schema mismatch can cause insert failure
- **Sub-workflow reference:** None

### Return Response
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; sends final JSON response to the original caller.
- **Configuration choices:**  
  - Response type: JSON
  - Response body:
    - `result` from `$json.output`
    - `latency` from `$json.executionLatencyMs`
    - `model` from `$json.modelUsed`
    - `cost` from `$json.estimatedCost`
- **Key expressions or variables used:**  
  - `$json.output`
  - `$json.executionLatencyMs`
  - `$json.modelUsed`
  - `$json.estimatedCost`
- **Input and output connections:**  
  - Input from `Store Telemetry`
  - Final responder for webhook flow
- **Version-specific requirements:** Type version `1.5`
- **Edge cases or potential failure types:**  
  - If `Store Telemetry` outputs only DB insert metadata, the original `output` field may be lost
  - If telemetry fields are not preserved after DB insert, the response may contain nulls
  - Response template is not quoting injected values as strings, so malformed JSON is possible if `output` is plain text containing quotes/newlines and not already JSON-safe
- **Sub-workflow reference:** None

---

## 2.6 Weekly Self-Tuning Optimization

**Overview:**  
This secondary branch runs on a schedule and attempts to optimize model routing automatically based on telemetry collected during the past week. It is effectively a maintenance and policy-learning loop.

**Nodes Involved:**  
- Weekly Self-Tuning Schedule  
- Fetch Historical Data  
- Aggregate Success Metrics  
- Calculate Routing Adjustments  
- Update Policy Rules

### Weekly Self-Tuning Schedule
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; time-based trigger.
- **Configuration choices:**  
  - Runs every week at hour `2`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Entry point for optimization branch
  - Outputs to `Fetch Historical Data`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**  
  - n8n instance timezone affects exact execution time
  - Missed runs if workflow is inactive or server unavailable
- **Sub-workflow reference:** None

### Fetch Historical Data
- **Type and technical role:** `n8n-nodes-base.postgres`; queries 7-day telemetry aggregates by task type and model.
- **Configuration choices:**  
  - SQL computes:
    - `AVG(latency_ms)` as `avg_latency`
    - `AVG(estimated_cost)` as `avg_cost`
    - `COUNT(*)` as `total_executions`
    - successful execution count
  - Filters to last 7 days
  - Groups by `task_type`, `model_used`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from `Weekly Self-Tuning Schedule`
  - Output to `Aggregate Success Metrics`
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**  
  - Empty telemetry table yields no output
  - Numeric aggregates may come back as strings depending on Postgres driver/settings
- **Sub-workflow reference:** None

### Aggregate Success Metrics
- **Type and technical role:** `n8n-nodes-base.aggregate`; aggregates incoming records.
- **Configuration choices:**  
  - `aggregateAllItemData`
  - Includes specified field: `task_type`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from `Fetch Historical Data`
  - Output to `Calculate Routing Adjustments`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**  
  - The output shape may become an array/object bundle rather than one item per metric
  - This matters because the next Code node expects iterable `metrics`
- **Sub-workflow reference:** None

### Calculate Routing Adjustments
- **Type and technical role:** `n8n-nodes-base.code`; converts historical metrics into model recommendations.
- **Configuration choices:**  
  - Reads `const metrics = $input.first().json;`
  - Iterates over `metrics`
  - Rules:
    - If current model is `large`, success rate > 0.95, and latency < 5000 → recommend `small`
    - If current model is `small` and success rate < 0.80 → recommend `large`
    - Else keep current model
  - Returns array of adjustment objects with:
    - `task_type`
    - `current_model`
    - `recommended_model`
    - `success_rate`
    - `avg_latency`
    - `avg_cost`
    - `adjustment_reason`
- **Key expressions or variables used:**  
  - `$input.first().json`
- **Input and output connections:**  
  - Input from `Aggregate Success Metrics`
  - Output to `Update Policy Rules`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**  
  - Potential shape mismatch:
    - if aggregate node outputs an object that is not directly iterable, `for (const metric of metrics)` fails
  - Division by zero if `total_executions` is zero
  - Numeric strings from SQL may require parsing
- **Sub-workflow reference:** None

### Update Policy Rules
- **Type and technical role:** `n8n-nodes-base.postgres`; writes optimized preferred model values back to policy table.
- **Configuration choices:**  
  - Operation: `executeQuery`
  - SQL:
    - `UPDATE policy_rules SET preferred_model = $1 WHERE task_type = $2`
  - Query replacements:
    - `={{ $json.recommended_model }}`
    - `={{ $json.task_type }}`
- **Key expressions or variables used:**  
  - `$json.recommended_model`
  - `$json.task_type`
- **Input and output connections:**  
  - Input from `Calculate Routing Adjustments`
  - No downstream node connected
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**  
  - Updates all rows of a task type regardless of priority
  - If policy table has multiple priorities, this may unintentionally overwrite all of them
  - DB permission issues or SQL failure
- **Sub-workflow reference:** None

---

## 2.7 Sticky Notes / Embedded Documentation

**Overview:**  
These nodes provide visual documentation inside the canvas. They do not execute but are useful for understanding intent and design.

**Nodes Involved:**  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6  
- Sticky Note7  
- Sticky Note8  
- Sticky Note9  
- Sticky Note10  
- Sticky Note11  
- Sticky Note12

### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation
- **Configuration choices:** Describes the policy engine’s role in combining policies with classification and determining model, token, latency, and cost settings.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Sticky Note2
- Weekly optimization trigger explanation.

### Sticky Note3
- Input webhook/API entry point explanation.

### Sticky Note4
- Performance aggregation explanation.

### Sticky Note5
- Historical telemetry analysis explanation.

### Sticky Note6
- Automatic policy optimization explanation.

### Sticky Note7
- Telemetry collection explanation.

### Sticky Note8
- API response explanation.

### Sticky Note9
- Large model execution explanation.

### Sticky Note10
- Model routing explanation.

### Sticky Note11
- Small model execution explanation.

### Sticky Note12
- Full workflow summary and setup guidance.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Task Input Webhook | webhook | Main HTTP entry point for task requests |  | Workflow Configuration | ## Input\nEach request contains a task and optional priority metadata.\nThis webhook acts as the API entry point for the LLM orchestration system. |
| Workflow Configuration | set | Adds default latency, cost, timeout, token, and retry settings | Task Input Webhook | Task Classifier Agent | ## Policy-Driven LLM Orchestrator with Self-Tuning Model Routing\nThis workflow acts as an intelligent orchestration layer for large language models. Instead of sending every task to a single model, it dynamically decides which model to use based on task complexity, policies, and historical performance.\n\n## How it works\nA webhook receives tasks and sends them to a classifier agent that determines the task type (extraction, classification, reasoning, or generation) along with a confidence score.\nThe workflow then loads policy rules from a database and applies a policy engine to determine the optimal execution strategy. These rules define model size, latency limits, token budgets, retry behavior, and cost ceilings.\nTasks are routed to either a small or large model depending on complexity and priority.\nTelemetry such as latency, tokens used, estimated cost, and success status is stored for every execution.\nA weekly optimization workflow analyzes this telemetry data to adjust routing policies automatically, improving cost efficiency and performance over time.\n\n## Setup steps\n1. Connect a Postgres database.\n2. Create tables: policy_rules and telemetry.\n3. Add Anthropic credentials for the LLM nodes.\n4. Configure latency, cost, and token limits in the configuration node.\n5. Send tasks to the webhook endpoint. |
| Task Classifier Agent | langchain agent | Classifies task type and confidence | Workflow Configuration | Load Policy Rules | ## Policy Engine\nLoads routing policies from the database and combines them with task classification results.\n\nThe policy engine determines:\n\n• model size (small or large)\n• token limits\n• latency constraints\n• cost budget |
| Anthropic Chat Model - Classifier | Anthropic chat model | LLM backend for classification |  | Task Classifier Agent | ## Policy Engine\nLoads routing policies from the database and combines them with task classification results.\n\nThe policy engine determines:\n\n• model size (small or large)\n• token limits\n• latency constraints\n• cost budget |
| Classification Output Parser | structured output parser | Enforces classifier JSON schema |  | Task Classifier Agent | ## Policy Engine\nLoads routing policies from the database and combines them with task classification results.\n\nThe policy engine determines:\n\n• model size (small or large)\n• token limits\n• latency constraints\n• cost budget |
| Load Policy Rules | postgres | Retrieves matching policy rule from DB | Task Classifier Agent | Policy Engine Decision | ## Policy Engine\nLoads routing policies from the database and combines them with task classification results.\n\nThe policy engine determines:\n\n• model size (small or large)\n• token limits\n• latency constraints\n• cost budget |
| Policy Engine Decision | code | Merges defaults, classification, and policy into routing decision | Load Policy Rules | Route by Model Selection | ## Policy Engine\nLoads routing policies from the database and combines them with task classification results.\n\nThe policy engine determines:\n\n• model size (small or large)\n• token limits\n• latency constraints\n• cost budget |
| Route by Model Selection | switch | Branches to small or large model path | Policy Engine Decision | Small Model Execution, Large Model Execution | ## Model Routing\nTasks are routed to different model pipelines depending on policy decisions. |
| Small Model Execution | langchain agent | Executes lightweight tasks | Route by Model Selection | Prepare Telemetry Data | ## Small Model Execution\nHandles lightweight tasks such as extraction or classification.\n\nOptimized for speed and cost efficiency while respecting token and latency limits. |
| Anthropic Chat Model - Small | Anthropic chat model | LLM backend for small model path |  | Small Model Execution | ## Small Model Execution\nHandles lightweight tasks such as extraction or classification.\n\nOptimized for speed and cost efficiency while respecting token and latency limits. |
| Large Model Execution | langchain agent | Executes complex/high-priority tasks | Route by Model Selection | Prepare Telemetry Data | ## Large Model Execution\nProcesses complex tasks that require deeper reasoning or higher accuracy.\n\nUsed when the policy engine detects low confidence, high priority, or reasoning tasks. |
| Anthropic Chat Model - Large | Anthropic chat model | LLM backend for large model path |  | Large Model Execution | ## Large Model Execution\nProcesses complex tasks that require deeper reasoning or higher accuracy.\n\nUsed when the policy engine detects low confidence, high priority, or reasoning tasks. |
| Prepare Telemetry Data | set | Adds execution metrics before persistence | Small Model Execution, Large Model Execution | Store Telemetry | ## Telemetry Collection\nExecution metrics are captured after each task:\n\n• latency\n• tokens used\n• estimated cost\n• model used\n• success status\n\nThese metrics are stored for monitoring and optimization. |
| Store Telemetry | postgres | Persists telemetry into Postgres | Prepare Telemetry Data | Return Response | ## Telemetry Collection\nExecution metrics are captured after each task:\n\n• latency\n• tokens used\n• estimated cost\n• model used\n• success status\n\nThese metrics are stored for monitoring and optimization. |
| Return Response | respondToWebhook | Returns AI result plus metadata to caller | Store Telemetry |  | API Response\nThe workflow returns the AI response along with execution metadata including latency, model used, and estimated cost. |
| Weekly Self-Tuning Schedule | scheduleTrigger | Weekly entry point for optimization |  | Fetch Historical Data | ## Weekly Optimization Trigger\nThis scheduled trigger runs once per week to analyze LLM execution performance. |
| Fetch Historical Data | postgres | Reads 7-day telemetry aggregates | Weekly Self-Tuning Schedule | Aggregate Success Metrics | ## Historical Telemetry Analysis\nExecution telemetry from the past 7 days is retrieved from the database. |
| Aggregate Success Metrics | aggregate | Aggregates telemetry records for optimization | Fetch Historical Data | Calculate Routing Adjustments | ## Performance Aggregation\nTelemetry records are grouped by task type to calculate overall performance metrics. |
| Calculate Routing Adjustments | code | Produces policy recommendations from metrics | Aggregate Success Metrics | Update Policy Rules | ## Automatic Policy Optimization\nModel performance is analyzed to determine if routing policies should change.\nIf a smaller model performs well, it may replace a larger one. |
| Update Policy Rules | postgres | Updates preferred model in policy table | Calculate Routing Adjustments |  | ## Automatic Policy Optimization\nModel performance is analyzed to determine if routing policies should change.\nIf a smaller model performs well, it may replace a larger one. |
| Sticky Note1 | stickyNote | Visual documentation for policy engine |  |  | ## Policy Engine\nLoads routing policies from the database and combines them with task classification results.\n\nThe policy engine determines:\n\n• model size (small or large)\n• token limits\n• latency constraints\n• cost budget |
| Sticky Note2 | stickyNote | Visual documentation for weekly trigger |  |  | ## Weekly Optimization Trigger\nThis scheduled trigger runs once per week to analyze LLM execution performance. |
| Sticky Note3 | stickyNote | Visual documentation for input block |  |  | ## Input\nEach request contains a task and optional priority metadata.\nThis webhook acts as the API entry point for the LLM orchestration system. |
| Sticky Note4 | stickyNote | Visual documentation for performance aggregation |  |  | ## Performance Aggregation\nTelemetry records are grouped by task type to calculate overall performance metrics. |
| Sticky Note5 | stickyNote | Visual documentation for historical telemetry |  |  | ## Historical Telemetry Analysis\nExecution telemetry from the past 7 days is retrieved from the database. |
| Sticky Note6 | stickyNote | Visual documentation for policy optimization |  |  | ## Automatic Policy Optimization\nModel performance is analyzed to determine if routing policies should change.\nIf a smaller model performs well, it may replace a larger one. |
| Sticky Note7 | stickyNote | Visual documentation for telemetry collection |  |  | ## Telemetry Collection\nExecution metrics are captured after each task:\n\n• latency\n• tokens used\n• estimated cost\n• model used\n• success status\n\nThese metrics are stored for monitoring and optimization. |
| Sticky Note8 | stickyNote | Visual documentation for API response |  |  | API Response\nThe workflow returns the AI response along with execution metadata including latency, model used, and estimated cost. |
| Sticky Note9 | stickyNote | Visual documentation for large-model path |  |  | ## Large Model Execution\nProcesses complex tasks that require deeper reasoning or higher accuracy.\n\nUsed when the policy engine detects low confidence, high priority, or reasoning tasks. |
| Sticky Note10 | stickyNote | Visual documentation for routing |  |  | ## Model Routing\nTasks are routed to different model pipelines depending on policy decisions. |
| Sticky Note11 | stickyNote | Visual documentation for small-model path |  |  | ## Small Model Execution\nHandles lightweight tasks such as extraction or classification.\n\nOptimized for speed and cost efficiency while respecting token and latency limits. |
| Sticky Note12 | stickyNote | Visual documentation for overall architecture |  |  | ## Policy-Driven LLM Orchestrator with Self-Tuning Model Routing\nThis workflow acts as an intelligent orchestration layer for large language models. Instead of sending every task to a single model, it dynamically decides which model to use based on task complexity, policies, and historical performance.\n\n## How it works\nA webhook receives tasks and sends them to a classifier agent that determines the task type (extraction, classification, reasoning, or generation) along with a confidence score.\nThe workflow then loads policy rules from a database and applies a policy engine to determine the optimal execution strategy. These rules define model size, latency limits, token budgets, retry behavior, and cost ceilings.\nTasks are routed to either a small or large model depending on complexity and priority.\nTelemetry such as latency, tokens used, estimated cost, and success status is stored for every execution.\nA weekly optimization workflow analyzes this telemetry data to adjust routing policies automatically, improving cost efficiency and performance over time.\n\n## Setup steps\n1. Connect a Postgres database.\n2. Create tables: policy_rules and telemetry.\n3. Add Anthropic credentials for the LLM nodes.\n4. Configure latency, cost, and token limits in the configuration node.\n5. Send tasks to the webhook endpoint. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence in n8n.

## Prerequisites
1. Create Anthropic API credentials in n8n for LangChain Anthropic chat model nodes.
2. Create Postgres credentials in n8n.
3. Create these database tables before activating the workflow.

### Suggested `policy_rules` table
Use columns at minimum:
- `task_type` text
- `priority` text
- `preferred_model` text
- `allow_tools` boolean
- `retry_strategy` text
- `max_latency_ms` integer
- `max_tokens` integer
- `cost_limit` numeric

### Suggested `telemetry` table
Use columns at minimum:
- `task_type` text
- `priority` text
- `model_used` text
- `latency_ms` numeric
- `tokens_used` numeric
- `estimated_cost` numeric
- `success` boolean
- `failure_type` text
- `timestamp` timestamptz

## Build steps

1. **Create a Webhook node**
   - Name: `Task Input Webhook`
   - Method: `POST`
   - Path: `llm-orchestrator`
   - Response mode: `Last Node`
   - Expected request body should include at least:
     - `task`
     - optionally `priority`

2. **Create a Set node**
   - Name: `Workflow Configuration`
   - Enable “Include Other Input Fields”
   - Add numeric fields:
     - `maxLatencyMs = 30000`
     - `costCeiling = 0.5`
     - `timeoutMs = 25000`
     - `maxTokens = 4000`
     - `maxRetries = 2`
   - Connect `Task Input Webhook -> Workflow Configuration`

3. **Create an Anthropic Chat Model node**
   - Name: `Anthropic Chat Model - Classifier`
   - Choose Anthropic credentials
   - Model: `claude-3-5-haiku-20241022`

4. **Create a Structured Output Parser node**
   - Name: `Classification Output Parser`
   - Schema type: manual
   - Define schema with:
     - `taskType` string
     - `confidence` number
     - `priority` string

5. **Create a LangChain Agent node**
   - Name: `Task Classifier Agent`
   - Text input: `{{ $json.task }}`
   - Prompt type: define
   - System message should instruct the model to classify tasks into:
     - extraction
     - classification
     - reasoning
     - generation
   - Require a confidence score from 0 to 100
   - Enable output parser
   - Connect:
     - `Workflow Configuration -> Task Classifier Agent`
     - `Anthropic Chat Model - Classifier -> Task Classifier Agent` via AI language model port
     - `Classification Output Parser -> Task Classifier Agent` via AI output parser port

6. **Create a Postgres node for policy lookup**
   - Name: `Load Policy Rules`
   - Operation: Execute Query
   - SQL:
     - `SELECT * FROM policy_rules WHERE task_type = $1 AND priority = $2`
   - Query replacements:
     - first replacement: `{{ $json.taskType }}`
     - second replacement: `{{ $json.priority }}`
   - Connect `Task Classifier Agent -> Load Policy Rules`

7. **Create a Code node**
   - Name: `Policy Engine Decision`
   - Purpose: merge config defaults, classifier output, and policy row
   - Recreate the decision object with:
     - `modelSize`
     - `allowTools`
     - `retryStrategy`
     - `timeoutMs`
     - `maxTokens`
     - `costBudget`
     - `taskType`
     - `confidence`
     - `priority`
   - Add override rules:
     - reasoning + confidence < 70 => `large`
     - priority = `critical` => `large`
   - **Recommended fix while rebuilding:** reference the classifier explicitly:
     - `const config = $('Workflow Configuration').first().json;`
     - `const classification = $('Task Classifier Agent').first().json;`
     - `const policy = $json;`
   - Also explicitly preserve original `task` by adding `task: $('Workflow Configuration').first().json.task`
   - Connect `Load Policy Rules -> Policy Engine Decision`

8. **Create a Switch node**
   - Name: `Route by Model Selection`
   - Value to evaluate: `{{ $json.modelSize }}`
   - Add condition/output for `small`
   - Add condition/output for `large`
   - Optional fallback output
   - Connect `Policy Engine Decision -> Route by Model Selection`

9. **Create an Anthropic Chat Model node**
   - Name: `Anthropic Chat Model - Small`
   - Credentials: Anthropic
   - Model: `claude-3-5-haiku-20241022`

10. **Create a LangChain Agent node**
    - Name: `Small Model Execution`
    - Text input: `{{ $json.task }}`
    - System message should describe a fast, efficient assistant
    - Mention `{{ $json.taskType }}` and `{{ $json.priority }}`
    - Connect:
      - `Route by Model Selection (small) -> Small Model Execution`
      - `Anthropic Chat Model - Small -> Small Model Execution`

11. **Create an Anthropic Chat Model node**
    - Name: `Anthropic Chat Model - Large`
    - Credentials: Anthropic
    - Model: `claude-3-5-sonnet-20241022`

12. **Create a LangChain Agent node**
    - Name: `Large Model Execution`
    - Text input: `{{ $json.task }}`
    - System message should describe a more advanced assistant for complex reasoning
    - Mention `{{ $json.taskType }}` and `{{ $json.priority }}`
    - Connect:
      - `Route by Model Selection (large) -> Large Model Execution`
      - `Anthropic Chat Model - Large -> Large Model Execution`

13. **Create a Set node for telemetry**
    - Name: `Prepare Telemetry Data`
    - Include other input fields
    - Add fields such as:
      - `executionLatencyMs`
      - `modelUsed`
      - `tokensUsed`
      - `estimatedCost`
      - `success = true`
      - `failureType = null or empty string`
      - `timestamp = {{ $now.toISO() }}`
    - **Recommended fix while rebuilding:** store snake_case fields too, to match DB:
      - `task_type = {{ $json.taskType }}`
      - `model_used = {{ $json.modelSize }}`
      - `latency_ms = ...`
      - `tokens_used = ...`
      - `estimated_cost = ...`
      - `failure_type = ...`
    - Connect both:
      - `Small Model Execution -> Prepare Telemetry Data`
      - `Large Model Execution -> Prepare Telemetry Data`

14. **Create a Postgres node to insert telemetry**
    - Name: `Store Telemetry`
    - Operation: Insert or equivalent table-insert mode
    - Schema: `public`
    - Table: `telemetry`
    - Map columns:
      - `task_type`
      - `priority`
      - `model_used`
      - `latency_ms`
      - `tokens_used`
      - `estimated_cost`
      - `success`
      - `failure_type`
      - `timestamp`
    - Connect `Prepare Telemetry Data -> Store Telemetry`

15. **Create a Respond to Webhook node**
    - Name: `Return Response`
    - Response format: JSON
    - Return:
      - `result`
      - `latency`
      - `model`
      - `cost`
    - **Recommended fix while rebuilding:** respond from fields preserved before the Postgres insert, or configure the Postgres node so original fields remain accessible
    - Connect `Store Telemetry -> Return Response`

16. **Create a Schedule Trigger node**
    - Name: `Weekly Self-Tuning Schedule`
    - Frequency: weekly
    - Trigger hour: `2`

17. **Create a Postgres node for historical telemetry**
    - Name: `Fetch Historical Data`
    - Operation: Execute Query
    - Use query that aggregates the last 7 days by `task_type` and `model_used`
    - Connect `Weekly Self-Tuning Schedule -> Fetch Historical Data`

18. **Create an Aggregate node**
    - Name: `Aggregate Success Metrics`
    - Configure to aggregate all incoming item data
    - Include `task_type`
    - Connect `Fetch Historical Data -> Aggregate Success Metrics`

19. **Create a Code node**
    - Name: `Calculate Routing Adjustments`
    - Implement logic to compute:
      - success rate
      - latency
      - cost
      - recommended model
    - Rule set:
      - large -> small if success rate > 95% and latency < 5000
      - small -> large if success rate < 80%
    - Connect `Aggregate Success Metrics -> Calculate Routing Adjustments`
    - **Recommended fix while rebuilding:** verify the exact aggregate output shape; if needed, iterate over an array field instead of `$input.first().json`

20. **Create a Postgres node for policy updates**
    - Name: `Update Policy Rules`
    - Operation: Execute Query
    - SQL:
      - `UPDATE policy_rules SET preferred_model = $1 WHERE task_type = $2`
    - Replacements:
      - `{{ $json.recommended_model }}`
      - `{{ $json.task_type }}`
    - Connect `Calculate Routing Adjustments -> Update Policy Rules`

21. **Optional but recommended hardening**
    - Add validation for missing `task`
    - Add fallback behavior when no policy row exists
    - Add dedicated error handling branch
    - Normalize field names to either camelCase or snake_case throughout
    - Add priority default, for example `normal`
    - Pass timeout/token settings into model node options if supported by your installed n8n version

## Required credentials
- **Anthropic credentials**
  - Needed by:
    - `Anthropic Chat Model - Classifier`
    - `Anthropic Chat Model - Small`
    - `Anthropic Chat Model - Large`
- **Postgres credentials**
  - Needed by:
    - `Load Policy Rules`
    - `Store Telemetry`
    - `Fetch Historical Data`
    - `Update Policy Rules`

## Input expectations
Main webhook body should ideally look like:
- `task`: string, required
- `priority`: string, optional but recommended

## Output expectations
Main webhook returns:
- `result`
- `latency`
- `model`
- `cost`

No sub-workflows are used in this design.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow has two entry points: one webhook for real-time orchestration and one weekly schedule for policy optimization. | Architecture note |
| The current JSON contains several likely implementation mismatches: classifier data propagation into the policy engine, camelCase vs snake_case telemetry field names, and response-field preservation after the Postgres insert. | Important build/review note |
| The policy optimizer updates `preferred_model` by `task_type` only, not by `priority`. If policy rows are priority-specific, consider adding priority to the update condition. | Database design note |
| The small and large execution nodes describe token and latency budgets in prompts, but the workflow does not strictly enforce those budgets in model parameters. | Operational note |
| Sticky note summary includes setup guidance: connect Postgres, create `policy_rules` and `telemetry`, add Anthropic credentials, configure limits, and send tasks to the webhook endpoint. | Embedded project note |