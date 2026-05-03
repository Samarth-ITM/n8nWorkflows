Route AI queries cost‑efficiently with GPT‑4o‑mini, GPT‑4o and confidence scoring

https://n8nworkflows.xyz/workflows/route-ai-queries-cost-efficiently-with-gpt-4o-mini--gpt-4o-and-confidence-scoring-13966


# Route AI queries cost‑efficiently with GPT‑4o‑mini, GPT‑4o and confidence scoring

# 1. Workflow Overview

This workflow implements a **cost-aware AI response routing pattern**. It first answers a user query with a lower-cost OpenAI model (`gpt-4o-mini`), then uses a second AI-based evaluator to judge the quality of that answer. If the evaluator determines the answer is not good enough, the workflow escalates the same query to a stronger but more expensive model (`gpt-4o`). It then estimates token cost and returns a structured final response.

Typical use cases:

- AI assistants where cost control matters
- Support or FAQ systems that only escalate difficult questions
- Internal tools that need a balance between quality and API spend
- Agentic systems that need a confidence gate before using premium models

## 1.1 Input Reception and Runtime Configuration

The workflow starts from an HTTP webhook and enriches the incoming payload with runtime configuration values such as the escalation threshold and model pricing.

## 1.2 First-Pass Low-Cost Answer Generation

The incoming query is sent to a cheaper OpenAI chat model to generate the initial answer.

## 1.3 Response Quality Evaluation

A LangChain agent evaluates the cheap model’s response using a separate language model and a structured JSON parser. It outputs a confidence score, rationale, and escalation recommendation.

## 1.4 Escalation Decision

An IF node checks whether escalation is recommended. If yes, the original query is sent to the expensive model. If not, the workflow skips escalation.

## 1.5 Cost Estimation and Final Output Formatting

A Code node computes token-based estimated cost and cost difference between the cheap and expensive paths. A final Set node formats the workflow’s output payload.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Runtime Configuration

### Overview

This block receives the user request via HTTP POST and defines configuration values used later in the workflow. It serves as the entry point and central source for threshold and pricing values.

### Nodes Involved

- `Webhook Trigger`
- `Workflow Configuration`

### Node Details

#### Webhook Trigger

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Acts as the external HTTP entry point for the workflow.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: a generated webhook path ID
  - No extra options configured
- **Key expressions or variables used:**
  - No expressions in this node
  - Expects incoming payload fields, especially `query`
- **Input and output connections:**
  - No input node; this is the workflow trigger
  - Output goes to `Workflow Configuration`
- **Version-specific requirements:**
  - Uses `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Request body may not contain `query`
  - Consumer may call the wrong webhook URL or wrong method
  - If the workflow is inactive in production, production webhook calls will fail
- **Sub-workflow reference:**
  - None

#### Workflow Configuration

- **Type and technical role:** `n8n-nodes-base.set`  
  Adds shared configuration fields to the incoming item while preserving existing input data.
- **Configuration choices:**
  - `includeOtherFields: true`, so original request fields remain available
  - Adds:
    - `confidenceThreshold = 0.7`
    - `cheapModelCostPer1kTokens = 0.00015`
    - `expensiveModelCostPer1kTokens = 0.005`
- **Key expressions or variables used:**
  - No dynamic expressions in the configured assignments
- **Input and output connections:**
  - Input from `Webhook Trigger`
  - Output to `Cheap Model (GPT-5-mini)`
- **Version-specific requirements:**
  - Uses `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - The later Code node expects differently named fields (`confidence_threshold`, `cheap_model_cost_per_1k_tokens`, etc.), but also includes hardcoded fallbacks. This naming mismatch can produce incorrect escalation cost logic unless corrected.
  - If the incoming request lacks `query`, downstream AI nodes will receive empty input
- **Sub-workflow reference:**
  - None

---

## 2.2 First-Pass Low-Cost Answer Generation

### Overview

This block sends the incoming query to the cheaper OpenAI model to generate the initial answer. It is the primary cost-saving mechanism in the workflow.

### Nodes Involved

- `Cheap Model (GPT-5-mini)`

### Node Details

#### Cheap Model (GPT-5-mini)

- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Calls the OpenAI Responses-style model node to produce the initial answer.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Prompt content: `{{ $json.query }}`
  - No additional options or tools configured
- **Key expressions or variables used:**
  - `={{ $json.query }}`
- **Input and output connections:**
  - Input from `Workflow Configuration`
  - Output to `Confidence Evaluator`
- **Version-specific requirements:**
  - Uses `typeVersion 2.1`
  - Requires valid OpenAI credentials in n8n
- **Edge cases or potential failure types:**
  - Missing `query` leads to poor or empty responses
  - OpenAI authentication or quota errors
  - Model availability or rate limiting issues
  - Response shape can vary across node versions; downstream nodes assume `message.content` and `usage.total_tokens`
- **Sub-workflow reference:**
  - None

---

## 2.3 Response Quality Evaluation

### Overview

This block evaluates whether the cheap model’s answer is good enough. It uses a LangChain agent powered by a separate OpenAI chat model and a structured parser to produce machine-readable confidence output.

### Nodes Involved

- `Confidence Evaluator`
- `OpenAI Chat Model`
- `Parse Confidence JSON`

### Node Details

#### Confidence Evaluator

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Runs an agent that evaluates the cheap response against the original query.
- **Configuration choices:**
  - Prompt type: explicitly defined
  - Input text includes:
    - original query from `Workflow Configuration`
    - cheap model response from current input item
  - System message instructs the model to assess:
    1. Accuracy
    2. Completeness
    3. Clarity
    4. Relevance
  - The node is configured with an output parser
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.query }}`
  - `{{ $json.message.content }}`
- **Input and output connections:**
  - Main input from `Cheap Model (GPT-5-mini)`
  - AI language model input from `OpenAI Chat Model`
  - AI output parser input from `Parse Confidence JSON`
  - Main output to `Check Confidence Threshold`
- **Version-specific requirements:**
  - Uses `typeVersion 3`
  - Requires LangChain-compatible n8n setup
- **Edge cases or potential failure types:**
  - If the cheap model output does not expose `message.content`, evaluation prompt may be malformed
  - LLM may still produce output that does not fully match schema expectations
  - Prompt quality impacts consistency of confidence scoring
- **Sub-workflow reference:**
  - None

#### OpenAI Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the language model used by the confidence evaluation agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - No additional options or tools
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - No main input
  - Connected via `ai_languageModel` to `Confidence Evaluator`
- **Version-specific requirements:**
  - Uses `typeVersion 1.3`
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - Auth, quota, timeout, or rate limit errors
  - Model name support depends on current n8n/OpenAI node compatibility
- **Sub-workflow reference:**
  - None

#### Parse Confidence JSON

- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a structured schema for the evaluator’s output.
- **Configuration choices:**
  - Example JSON schema:
    - `confidence_score`
    - `reason`
    - `should_escalate`
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - No main input
  - Connected via `ai_outputParser` to `Confidence Evaluator`
- **Version-specific requirements:**
  - Uses `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - If the evaluator returns malformed JSON, parsing can fail
  - Type coercion issues may occur if the model returns strings instead of booleans/numbers
- **Sub-workflow reference:**
  - None

---

## 2.4 Escalation Decision

### Overview

This block decides whether to escalate the query to the more expensive model. It branches the workflow based on the evaluator’s recommendation.

### Nodes Involved

- `Check Confidence Threshold`
- `Expensive Model (GPT-5.4)`

### Node Details

#### Check Confidence Threshold

- **Type and technical role:** `n8n-nodes-base.if`  
  Branches execution depending on whether escalation is recommended.
- **Configuration choices:**
  - Condition checks whether `{{$json.should_escalate}}` is `true`
  - Loose type validation enabled
- **Key expressions or variables used:**
  - `={{ $json.should_escalate }}`
- **Input and output connections:**
  - Input from `Confidence Evaluator`
  - True output to `Expensive Model (GPT-5.4)`
  - False output to `Calculate Cost Difference`
- **Version-specific requirements:**
  - Uses `typeVersion 2.3`
- **Edge cases or potential failure types:**
  - This node expects `should_escalate` at the root of the incoming JSON
  - However, `Format Final Response` later references `output.confidence_score`, suggesting the actual evaluator output may be nested under `output`
  - If the parsed result is nested, this IF condition may never behave as intended unless the path is corrected
- **Sub-workflow reference:**
  - None

#### Expensive Model (GPT-5.4)

- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Generates a higher-quality answer when the cheap answer is deemed insufficient.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Prompt content pulls original query from `Workflow Configuration`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.query }}`
- **Input and output connections:**
  - Input from true branch of `Check Confidence Threshold`
  - Output to `Calculate Cost Difference`
- **Version-specific requirements:**
  - Uses `typeVersion 2.1`
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - OpenAI auth/quota/rate limit failures
  - If `query` is missing, escalation produces low-quality output
  - Node name suggests “GPT-5.4” but actual configured model is `gpt-4o`; this is a naming inconsistency that can confuse maintainers or AI agents
- **Sub-workflow reference:**
  - None

---

## 2.5 Cost Estimation and Final Output Formatting

### Overview

This block determines which model path was used, estimates token-based API cost, computes the difference versus the alternate path, and prepares the final structured output.

### Nodes Involved

- `Calculate Cost Difference`
- `Format Final Response`

### Node Details

#### Calculate Cost Difference

- **Type and technical role:** `n8n-nodes-base.code`  
  Uses JavaScript to aggregate data from multiple prior nodes and compute cost metrics.
- **Configuration choices:**
  - Execution mode: `runOnceForEachItem`
  - Reads:
    - evaluator output
    - workflow config
    - cheap model output
    - expensive model output if escalation occurred
- **Key expressions or variables used:**
  - `$('Confidence Evaluator').item.json`
  - `$('Workflow Configuration').first().json`
  - `$('Cheap Model (GPT-5-mini)').item.json`
  - `$('Expensive Model (GPT-5.4)').item.json`
- **Input and output connections:**
  - Input from:
    - false branch of `Check Confidence Threshold`, or
    - `Expensive Model (GPT-5.4)` after escalation
  - Output to `Format Final Response`
- **Version-specific requirements:**
  - Uses `typeVersion 2`
- **Edge cases or potential failure types:**
  - **Important field mismatch:** the script checks `confidenceData.confidence`, but the parser schema defines `confidence_score`
  - **Important config mismatch:** script expects snake_case fields like `confidence_threshold`, but `Workflow Configuration` creates camelCase fields like `confidenceThreshold`
  - Because of these mismatches, `wasEscalated` may be computed incorrectly
  - It returns `confidence_score: confidenceData.confidence`, which likely does not exist
  - It returns `response` correctly based on branch
  - It does **not** return `cost_analysis`, yet `Format Final Response` tries to read it
  - If the code runs on the non-escalated branch, it still references evaluator/config/cheap-model nodes directly; that is fine as long as those nodes executed for the current item
  - If response shapes change, `usage.total_tokens` or `message.content` may be undefined
- **Sub-workflow reference:**
  - None

#### Format Final Response

- **Type and technical role:** `n8n-nodes-base.set`  
  Produces the final response object returned by the workflow.
- **Configuration choices:**
  - Sets:
    - `response`
    - `model_used`
    - `confidence_score`
    - `escalated`
    - `cost_usd`
    - `cost_analysis`
  - It does not preserve an explicitly defined schema beyond these fields
- **Key expressions or variables used:**
  - Response:
    - `{{ $('Expensive Model (GPT-5.4)').item.json.message?.content || $('Cheap Model (GPT-5-mini)').item.json.message?.content }}`
  - Model:
    - `{{ $('Calculate Cost Difference').item.json.model_used }}`
  - Confidence:
    - `{{ $('Confidence Evaluator').item.json.output?.confidence_score }}`
  - Escalated:
    - `{{ $('Calculate Cost Difference').item.json.escalated }}`
  - Cost:
    - `{{ $('Calculate Cost Difference').item.json.cost_usd }}`
  - Cost analysis:
    - `{{ JSON.stringify($('Calculate Cost Difference').item.json.cost_analysis) }}`
- **Input and output connections:**
  - Input from `Calculate Cost Difference`
  - No downstream node configured
- **Version-specific requirements:**
  - Uses `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - `cost_analysis` is referenced but not produced by the Code node, so this field will likely become `"undefined"` or fail depending on runtime handling
  - If `Expensive Model (GPT-5.4)` never ran, direct reference may still resolve poorly in some execution contexts; the OR fallback is intended to handle this, but cross-branch item resolution should be validated
  - If evaluator output is not nested under `output`, confidence score expression will be empty
- **Sub-workflow reference:**
  - None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | n8n-nodes-base.webhook | HTTP entry point for incoming queries |  | Workflow Configuration | ## Webhook Trigger\nStarts the workflow when an external application sends a request. |
| Workflow Configuration | n8n-nodes-base.set | Adds threshold and pricing configuration to incoming data | Webhook Trigger | Cheap Model (GPT-5-mini) | ## Workflow Configuration\n\nDefines key configuration settings used across the workflow.\nIncludes the confidence threshold and token pricing |
| Cheap Model (GPT-5-mini) | @n8n/n8n-nodes-langchain.openAi | Generates the first low-cost answer | Workflow Configuration | Confidence Evaluator | ## Cheap Model (GPT-5-mini)\n\nProcesses the user query using a low-cost AI model.\nThis step provides an initial response while minimizing token costs. |
| Confidence Evaluator | @n8n/n8n-nodes-langchain.agent | Evaluates cheap-model answer quality and recommends escalation | Cheap Model (GPT-5-mini); OpenAI Chat Model; Parse Confidence JSON | Check Confidence Threshold | ## Confidence Evaluator\nAnalyzes the cheap model’s response to determine its quality.\nEvaluates accuracy, completeness, clarity, and relevance. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies the LLM used by the evaluator agent |  | Confidence Evaluator | ## Confidence Evaluator\nAnalyzes the cheap model’s response to determine its quality.\nEvaluates accuracy, completeness, clarity, and relevance. |
| Parse Confidence JSON | @n8n/n8n-nodes-langchain.outputParserStructured | Forces structured evaluator output |  | Confidence Evaluator | ## Confidence Evaluator\nAnalyzes the cheap model’s response to determine its quality.\nEvaluates accuracy, completeness, clarity, and relevance. |
| Check Confidence Threshold | n8n-nodes-base.if | Routes execution based on escalation recommendation | Confidence Evaluator | Expensive Model (GPT-5.4); Calculate Cost Difference | ## Check Confidence Threshold\nDecision node that determines whether the response quality is sufficient. |
| Expensive Model (GPT-5.4) | @n8n/n8n-nodes-langchain.openAi | Generates fallback high-quality answer | Check Confidence Threshold | Calculate Cost Difference | ## Expensive Model (GPT-5.4)\n\nHandles queries that require higher quality reasoning.\nTriggered only when the confidence score is below the threshold. |
| Calculate Cost Difference | n8n-nodes-base.code | Computes estimated cost and selected model | Check Confidence Threshold; Expensive Model (GPT-5.4) | Format Final Response | ## Calculate Cost Difference\n\nCalculates token usage and estimated cost of the request.\nDetermines which model was used and compares costs between the cheap and expensive models. |
| Format Final Response | n8n-nodes-base.set | Builds final structured payload | Calculate Cost Difference |  | ## Format Final Response\n\nPrepares the final structured output returned by the workflow. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual overview note |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for expensive model block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation for cost block |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation for evaluation block |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation for cheap model block |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation for config block |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual documentation for final formatting block |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual documentation for webhook block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a manual reconstruction process for n8n.

## 4.1 Create the trigger and configuration

1. **Create a `Webhook` node**
   - Name it `Webhook Trigger`
   - Set **HTTP Method** to `POST`
   - Set the **Path** to a unique endpoint path
   - Leave additional options at default unless you need custom auth or response mode
   - Expected request body should include at least:
     - `query` as a string

2. **Create a `Set` node**
   - Name it `Workflow Configuration`
   - Connect `Webhook Trigger -> Workflow Configuration`
   - Enable inclusion of other incoming fields
   - Add these fields:
     - `confidenceThreshold` as Number = `0.7`
     - `cheapModelCostPer1kTokens` as Number = `0.00015`
     - `expensiveModelCostPer1kTokens` as Number = `0.005`

   **Important recommendation:**  
   To avoid downstream field mismatch, either:
   - keep the exact names expected by your Code node, or
   - update later expressions/scripts to use these camelCase names consistently.

## 4.2 Create the cheap-answer generation step

3. **Create an OpenAI node of type `OpenAI` from the LangChain/OpenAI set**
   - Name it `Cheap Model (GPT-5-mini)`
   - Connect `Workflow Configuration -> Cheap Model (GPT-5-mini)`
   - Select OpenAI credentials
   - Set model to `gpt-4o-mini`
   - In the response input/content field, use:
     - `{{ $json.query }}`
   - Leave tools disabled unless you explicitly want tool use

## 4.3 Create the confidence evaluation block

4. **Create a LangChain Chat Model node**
   - Name it `OpenAI Chat Model`
   - Select OpenAI credentials
   - Set model to `gpt-4o-mini`

5. **Create a Structured Output Parser node**
   - Name it `Parse Confidence JSON`
   - Configure schema example as:
     ```json
     {
       "confidence_score": 0.85,
       "reason": "Brief explanation of the confidence assessment",
       "should_escalate": false
     }
     ```

6. **Create a LangChain Agent node**
   - Name it `Confidence Evaluator`
   - Connect:
     - `Cheap Model (GPT-5-mini) -> Confidence Evaluator` via main connection
     - `OpenAI Chat Model -> Confidence Evaluator` via AI language model connection
     - `Parse Confidence JSON -> Confidence Evaluator` via AI output parser connection
   - Set prompt type to defined/manual prompt
   - In the text prompt field, use:
     - `Original query: {{ $('Workflow Configuration').first().json.query }}`
     - `Cheap model response: {{ $json.message.content }}`
   - In the system message, instruct the evaluator to:
     - assess accuracy, completeness, clarity, and relevance
     - return a score from 0 to 1
     - explain the score briefly
     - decide whether escalation is needed
   - Enable output parser usage

   **Recommended consistency fix:**  
   Ensure the agent’s parsed output is exposed either:
   - directly at root as `confidence_score`, `reason`, `should_escalate`, or
   - nested under a predictable object, and then use that same structure everywhere.

## 4.4 Create the escalation branch

7. **Create an `If` node**
   - Name it `Check Confidence Threshold`
   - Connect `Confidence Evaluator -> Check Confidence Threshold`
   - Configure condition:
     - left value: parsed `should_escalate`
     - operation: `is true`

   **Recommended expression:**  
   Use whichever path matches your actual agent output, for example:
   - `{{ $json.should_escalate }}` if parser output is root-level
   - or `{{ $json.output.should_escalate }}` if nested

8. **Create another OpenAI node**
   - Name it `Expensive Model (GPT-5.4)`
   - Connect the **true** output of `Check Confidence Threshold` to this node
   - Select OpenAI credentials
   - Set model to `gpt-4o`
   - Use the original query as input:
     - `{{ $('Workflow Configuration').first().json.query }}`

   **Naming note:**  
   If you want the node label to match the actual model, consider renaming it to `Expensive Model (GPT-4o)`.

## 4.5 Create the cost calculation step

9. **Create a `Code` node**
   - Name it `Calculate Cost Difference`
   - Connect:
     - the **false** output of `Check Confidence Threshold` to `Calculate Cost Difference`
     - `Expensive Model (GPT-5.4)` to `Calculate Cost Difference`
   - Set mode to `Run Once for Each Item`

10. **Paste a corrected cost-calculation script**
   - The JSON workflow contains mismatched field names, so a corrected script is strongly recommended.
   - Use logic equivalent to:
     - read evaluator output
     - read config values
     - read cheap model usage
     - if escalated, read expensive model usage and response
     - compute selected model cost
     - compute delta versus the alternate path
     - return normalized fields

   A safe structure to implement:

   ```javascript
   const confidenceData = $('Confidence Evaluator').item.json;
   const configData = $('Workflow Configuration').first().json;
   const cheapModelData = $('Cheap Model (GPT-5-mini)').item.json;

   const parsed = confidenceData.output ?? confidenceData;

   const shouldEscalate = parsed.should_escalate === true;
   const confidenceScore = parsed.confidence_score ?? null;

   const cheapRate = configData.cheapModelCostPer1kTokens ?? 0.00015;
   const expensiveRate = configData.expensiveModelCostPer1kTokens ?? 0.005;

   let modelUsed, totalTokens, costUsd, costDifferenceUsd, response;

   const cheapTokens = cheapModelData.usage?.total_tokens ?? 0;
   const cheapCost = (cheapTokens / 1000) * cheapRate;

   if (shouldEscalate) {
     const expensiveModelData = $('Expensive Model (GPT-5.4)').item.json;
     const expensiveTokens = expensiveModelData.usage?.total_tokens ?? 0;
     const expensiveCost = (expensiveTokens / 1000) * expensiveRate;

     modelUsed = 'gpt-4o';
     totalTokens = expensiveTokens;
     costUsd = expensiveCost;
     costDifferenceUsd = expensiveCost - cheapCost;
     response = expensiveModelData.message?.content ?? null;
   } else {
     modelUsed = 'gpt-4o-mini';
     totalTokens = cheapTokens;
     costUsd = cheapCost;
     costDifferenceUsd = cheapCost - ((cheapTokens / 1000) * expensiveRate);
     response = cheapModelData.message?.content ?? null;
   }

   return {
     model_used: modelUsed,
     total_tokens: totalTokens,
     cost_usd: Number(costUsd.toFixed(6)),
     escalated: shouldEscalate,
     cost_difference_usd: Number(costDifferenceUsd.toFixed(6)),
     response,
     confidence_score: confidenceScore,
     cost_analysis: {
       cheap_model_cost_per_1k_tokens: cheapRate,
       expensive_model_cost_per_1k_tokens: expensiveRate
     }
   };
   ```

   This corrected version keeps naming consistent and ensures `cost_analysis` exists.

## 4.6 Create the final response formatter

11. **Create a `Set` node**
   - Name it `Format Final Response`
   - Connect `Calculate Cost Difference -> Format Final Response`

12. **Add output fields**
   - `response` → `{{ $('Calculate Cost Difference').item.json.response }}`
   - `model_used` → `{{ $('Calculate Cost Difference').item.json.model_used }}`
   - `confidence_score` → `{{ $('Calculate Cost Difference').item.json.confidence_score }}`
   - `escalated` → `{{ $('Calculate Cost Difference').item.json.escalated }}`
   - `cost_usd` → `{{ $('Calculate Cost Difference').item.json.cost_usd }}`
   - `cost_analysis` → `{{ $('Calculate Cost Difference').item.json.cost_analysis }}`
   - Optionally also include:
     - `cost_difference_usd`
     - `reason`

   **Recommendation:**  
   Prefer pulling all final fields from `Calculate Cost Difference` instead of directly referencing multiple earlier nodes. This makes branching more reliable.

## 4.7 Add response handling

13. **If you want the webhook to return the final JSON directly**
   - Configure the webhook response mode appropriately in your n8n version
   - Optionally add a `Respond to Webhook` node after `Format Final Response`
   - Return the structured payload

   The provided workflow JSON does not include a dedicated `Respond to Webhook` node, so depending on n8n settings you may need to add one for explicit API responses.

## 4.8 Configure credentials

14. **OpenAI credentials**
   - Create or select OpenAI credentials in n8n
   - Attach the same credential set to:
     - `Cheap Model (GPT-5-mini)`
     - `OpenAI Chat Model`
     - `Expensive Model (GPT-5.4)`

15. **Test with sample payload**
   - Send a POST request such as:
     ```json
     {
       "query": "Explain the difference between TCP and UDP."
     }
     ```
   - Validate:
     - cheap model responds
     - evaluator outputs valid JSON
     - IF node branches correctly
     - final node returns expected fields

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Cost-Optimized AI Model Routing: This workflow demonstrates a cost-aware AI routing strategy that balances response quality and API cost. Instead of always using an expensive model, the workflow first generates a response using a lower-cost AI model. A confidence evaluator then reviews the response quality. The workflow also estimates token usage and cost differences, helping measure the savings achieved through cost-optimized AI routing. How it works: Webhook Trigger receives a user query. A cheap AI model generates the initial response. A confidence evaluator agent analyzes the response quality. If confidence is low, the request is escalated to a stronger model. The workflow calculates token usage and cost difference. The final response and cost analysis are returned. | Overall workflow concept |
| Node names do not fully match actual models: `Cheap Model (GPT-5-mini)` uses `gpt-4o-mini`, and `Expensive Model (GPT-5.4)` uses `gpt-4o`. | Maintenance and clarity note |
| There are multiple field-structure inconsistencies in the JSON: camelCase config fields vs snake_case code references, `confidence_score` vs `confidence`, and missing `cost_analysis` in the Code node output. | Important implementation correction |
| No explicit `Respond to Webhook` node is present. Depending on webhook settings and desired API behavior, you may want to add one. | Deployment/runtime note |