Track LLM costs and usage across OpenAI, Anthropic, Google and more

https://n8nworkflows.xyz/workflows/track-llm-costs-and-usage-across-openai--anthropic--google-and-more-14536


# Track LLM costs and usage across OpenAI, Anthropic, Google and more

# 1. Workflow Overview

This workflow is a reusable **LLM execution cost monitor** for n8n. Its purpose is to inspect a completed n8n execution, detect LLM-related calls inside that execution, normalize model names, apply pricing rules, and output both **per-call cost details** and a **global usage summary**.

Typical use cases:
- Track OpenAI, Anthropic, Google Gemini, DeepSeek, Meta, Mistral, xAI, Cohere, Qwen, and Kimi usage
- Add cost observability to AI workflows without modifying each LLM node individually
- Build downstream reporting, dashboards, alerts, or historical storage from execution-level cost data

The workflow is designed primarily as a **sub-workflow** invoked from another workflow, though it also includes a disabled manual test path.

## 1.1 Input Reception and Execution ID Resolution

This block receives an execution ID either from another workflow or from a manual test node, then resolves the final execution ID value to inspect.

## 1.2 Execution Retrieval

This block uses the n8n API credential to fetch execution data for the target execution.

## 1.3 LLM Usage Extraction

This block scans the retrieved execution recursively, looking for token usage information across nested outputs, intermediate AI structures, and sub-node data.

## 1.4 LLM Node Metadata Discovery

This block identifies nodes related to LLM processing in the inspected workflow and extracts node-level metadata such as node type, position, previous-node chain, and execution timing.

## 1.5 Model Normalization and Validation

This block standardizes raw model names to canonical identifiers and checks whether every detected model is defined in the workflow’s internal mapping.

## 1.6 Error Handling for Undefined Models

If a model is not recognized, this block stops execution with an explicit error instructing the user to extend both the model-name mapping and the pricing dictionary.

## 1.7 Metadata Merge, Pricing, and Summary Output

This block merges usage data with node metadata, applies token pricing, then generates a final analytics object containing both detailed records and aggregated statistics.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Execution ID Resolution

### Overview
This block accepts the execution context from either a calling workflow or a manual test node. It ensures there is always an `executionId` field available before querying the n8n API.

### Nodes Involved
- When Called By Another Workflow
- Test with Execution ID
- Extract Execution ID

### Node Details

#### When Called By Another Workflow
- **Type and role:** `n8n-nodes-base.executeWorkflowTrigger`  
  Entry point for sub-workflow execution.
- **Configuration choices:**  
  Uses `inputSource: passthrough`, meaning the workflow receives the incoming payload from the parent workflow unchanged.
- **Key expressions or variables used:**  
  No custom expression on this node itself.
- **Input and output connections:**  
  No incoming connection; outputs to **Extract Execution ID**.
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases / failures:**  
  - Will only work when invoked by another workflow
  - Parent workflow must pass an execution ID or rely on fallback logic later
  - Subject to workflow caller policy restrictions
- **Sub-workflow reference:**  
  This workflow is intended to be called as a sub-workflow from another workflow.

#### Test with Execution ID
- **Type and role:** `n8n-nodes-base.code`  
  Disabled helper node for manual testing.
- **Configuration choices:**  
  Returns one item with a hardcoded `executionId` placeholder.
- **Key expressions or variables used:**  
  `const testExecutionId = 'PASTE_EXECUTION_ID_HERE';`
- **Input and output connections:**  
  No incoming connection; outputs to **Extract Execution ID**.
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases / failures:**  
  - Disabled by default
  - If enabled without replacing the placeholder, the next API lookup will fail
  - Useful only for manual debugging
- **Sub-workflow reference:**  
  None.

#### Extract Execution ID
- **Type and role:** `n8n-nodes-base.set`  
  Normalizes the incoming payload into a guaranteed `executionId` field.
- **Configuration choices:**  
  Creates one string field named `executionId`.
- **Key expressions or variables used:**  
  `{{ $json.executionId || $json.body?.executionId || $json.query?.executionId || $execution.id }}`
- **Input and output connections:**  
  Receives from **When Called By Another Workflow** or **Test with Execution ID**; outputs to **Get Execution Data**.
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases / failures:**  
  - If the payload lacks `executionId`, `body.executionId`, and `query.executionId`, it falls back to the current workflow’s own execution ID
  - That fallback can be misleading if the intent was to inspect a parent workflow execution rather than the monitor itself
- **Sub-workflow reference:**  
  None.

---

## 2.2 Execution Retrieval

### Overview
This block retrieves execution information using the n8n API credential. The rest of the workflow depends on this response containing full execution and workflow metadata.

### Nodes Involved
- Get Execution Data

### Node Details

#### Get Execution Data
- **Type and role:** `n8n-nodes-base.n8n`  
  n8n internal API access node, used here to retrieve execution-related data through an n8n credential context.
- **Configuration choices:**  
  - Resource: `credential`
  - Credential type name: `n8nApi`
  - Uses the configured n8n API credential named `n8n account 2`
- **Key expressions or variables used:**  
  No dynamic expression is visible in parameters, but functionally this node is intended to fetch execution data based on the previously resolved execution ID.
- **Input and output connections:**  
  Receives from **Extract Execution ID**; outputs in parallel to:
  - **Extract Token Usage**
  - **Find Nodes with LLM Data**
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases / failures:**  
  - Missing or invalid n8n API credential
  - Insufficient permission to query execution data
  - Execution ID not found
  - Self-hosted/cloud URL mismatch in credential setup
  - If execution payload does not include detailed run data, downstream extraction will return no LLM calls
- **Sub-workflow reference:**  
  None.

---

## 2.3 LLM Usage Extraction

### Overview
This block recursively scans execution results for token usage records. It is the main analytics extraction layer and attempts to detect usage regardless of nesting depth or provider-specific structure.

### Nodes Involved
- Extract Token Usage

### Node Details

#### Extract Token Usage
- **Type and role:** `n8n-nodes-base.code`  
  Deep parser for execution data that discovers LLM token usage inside run outputs and input overrides.
- **Configuration choices:**  
  The code:
  - Reads workflow name, workflow ID, execution ID, and status
  - Iterates over `executionData.data.resultData.runData`
  - Recursively explores nested objects/arrays
  - Looks for `tokenUsage` or `usage`
  - Extracts prompt, completion, and total tokens
  - Captures model name, finish reason, execution timing, previews, and selected generation settings
- **Key expressions or variables used:**  
  Internal code variables include:
  - `workflowName`
  - `workflowId`
  - `executionId`
  - `executionStatus`
  - `llmCalls`
  - recursive helper `extractTokenUsage(obj, nodeName, nodeType, path, depth)`
- **Input and output connections:**  
  Receives from **Get Execution Data**; outputs to **Standardize Names**.
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases / failures:**  
  - No LLM token usage found: returns one item with `error: 'No LLM calls detected in this execution'`
  - Recursion capped at depth 20, so very unusual deep nesting may be truncated
  - Ignores `binary` and `pairedItem` branches intentionally
  - Different providers may expose usage fields differently; unsupported structures may be missed
  - The expression for `executionStatus` is logically ambiguous because of operator precedence:
    `executionData.status || executionData.finished ? 'success' : 'error'`
    This may classify some executions unexpectedly
  - If execution payload shape changes in future n8n versions, extraction may break
- **Sub-workflow reference:**  
  None.

---

## 2.4 LLM Node Metadata Discovery

### Overview
This block identifies nodes that are LLM-related or at least executed in the target workflow, then builds metadata useful for analytics and traceability.

### Nodes Involved
- Find Nodes with LLM Data

### Node Details

#### Find Nodes with LLM Data
- **Type and role:** `n8n-nodes-base.code`  
  Discovers node-level context for executed LLM steps.
- **Configuration choices:**  
  The code:
  - Reads workflow nodes, run data, and connections
  - Defines a list of recognized LLM node types
  - Flags nodes as LLM-related by known type or type-name heuristics
  - Builds a reverse chain of previous nodes from workflow connections
  - Extracts execution time, start time, type, and position
- **Key expressions or variables used:**  
  Internal arrays/objects include:
  - `llmNodeTypes`
  - `workflowNodes`
  - `runData`
  - `connections`
  - recursive helper `findPrevious(nodeName, depth)`
- **Input and output connections:**  
  Receives from **Get Execution Data**; outputs to **Merge** as input 2.
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases / failures:**  
  - Includes nodes if they are recognized as LLM-related **or** if they appear in run data; this may include non-LLM executed nodes
  - Reverse dependency tracing is capped at depth 10
  - If node names are duplicated across versions or imported incorrectly, matching may be unreliable
  - If workflow connection structure differs, previous-node chain may be incomplete
- **Sub-workflow reference:**  
  None.

---

## 2.5 Model Normalization and Validation

### Overview
This block maps raw provider model names to canonical names and validates whether all models are known to the workflow. It is essential because pricing is keyed by standardized model identifiers.

### Nodes Involved
- Standardize Names
- All Models Defined?

### Node Details

#### Standardize Names
- **Type and role:** `n8n-nodes-base.code`  
  Converts raw model names into standardized identifiers.
- **Configuration choices:**  
  Uses a large in-node dictionary `standardize_names_dic` covering many model variants across multiple providers.
- **Key expressions or variables used:**  
  - `items = $input.all()`
  - `rawModel = (data.model || '').toLowerCase().trim()`
  - outputs:
    - `standardizedModel`
    - `modelKnown`
    - `rawModel`
- **Input and output connections:**  
  Receives from **Extract Token Usage**; outputs to **All Models Defined?**
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases / failures:**  
  - If upstream produced a single error item (`No LLM calls detected...`), it passes it through unchanged
  - Partial-match logic may map some unexpected names if a substring happens to match
  - Newly released models must be added manually
  - Empty or missing model names become `unknown` and `modelKnown = false`
- **Sub-workflow reference:**  
  None.

#### All Models Defined?
- **Type and role:** `n8n-nodes-base.if`  
  Branches based on whether each item has a recognized standardized model.
- **Configuration choices:**  
  Checks if `{{ $json.modelKnown }}` equals boolean `true`.
- **Key expressions or variables used:**  
  `={{ $json.modelKnown }}`
- **Input and output connections:**  
  Receives from **Standardize Names**.  
  True output goes to **Merge**.  
  False output goes to **Stop and Error**.
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases / failures:**  
  - Error-pass-through items from **Standardize Names** do not contain `modelKnown`; they will likely go to the false branch
  - Mixed batches with known and unknown models will split across both branches
- **Sub-workflow reference:**  
  None.

---

## 2.6 Error Handling for Undefined Models

### Overview
This block explicitly fails the workflow when unknown models are detected. It prevents silent cost calculation with incorrect mappings.

### Nodes Involved
- Stop and Error

### Node Details

#### Stop and Error
- **Type and role:** `n8n-nodes-base.code`  
  Throws a descriptive error listing all unknown model names.
- **Configuration choices:**  
  Collects all items where `modelKnown` is false, deduplicates them, and throws a formatted exception.
- **Key expressions or variables used:**  
  - `unknownModels`
  - `uniqueUnknown`
  - `throw new Error(...)`
- **Input and output connections:**  
  Receives from the false branch of **All Models Defined?**; no outgoing connection.
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases / failures:**  
  - Intentionally stops workflow execution
  - If upstream sent items with empty `rawModel`, the message may contain blank/unknown-like entries
  - This strict behavior is useful for governance but may be undesirable in production unless maintained regularly
- **Sub-workflow reference:**  
  None.

---

## 2.7 Metadata Merge, Pricing, and Summary Output

### Overview
This block combines usage items with node metadata, computes token costs using a built-in price list, and returns a final compact output containing detailed line items plus aggregated summary statistics.

### Nodes Involved
- Merge
- Model Prices
- Generate Summary

### Node Details

#### Merge
- **Type and role:** `n8n-nodes-base.merge`  
  Enriches the token-usage items with metadata from the node-discovery branch.
- **Configuration choices:**  
  - Mode: `combine`
  - Join mode: `enrichInput1`
  This means input 1 is the primary stream, enriched using input 2.
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  Input 1 from the true branch of **All Models Defined?**  
  Input 2 from **Find Nodes with LLM Data**  
  Outputs to **Model Prices**
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases / failures:**  
  - If the enrichment keys do not align as expected, metadata may not merge meaningfully
  - Depending on n8n merge semantics, mismatched item counts/order can produce incomplete enrichment
  - Since no explicit key-based join is configured, behavior depends on positional/item matching
- **Sub-workflow reference:**  
  None.

#### Model Prices
- **Type and role:** `n8n-nodes-base.code`  
  Applies per-million-token pricing to each standardized model.
- **Configuration choices:**  
  Contains a large `MODEL_PRICES` dictionary with input and output costs for many providers and models.
- **Key expressions or variables used:**  
  - `const model = data.standardizedModel || 'unknown'`
  - calculates:
    - `promptCost`
    - `completionCost`
    - `totalCost`
    - `inputPricePerM`
    - `outputPricePerM`
    - `pricingFound`
- **Input and output connections:**  
  Receives from **Merge**; outputs to **Generate Summary**
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases / failures:**  
  - If pricing is missing, fallback prices are used: input `1.00`, output `3.00`
  - Costs are rounded to 6 decimal places, which is appropriate for USD micro-costs but may still hide ultra-small values
  - Pricing must be maintained manually as providers update rates
- **Sub-workflow reference:**  
  None.

#### Generate Summary
- **Type and role:** `n8n-nodes-base.code`  
  Produces the final output object with summary analytics and detailed per-call records.
- **Configuration choices:**  
  Aggregates totals for:
  - cost
  - prompt/completion/total tokens
  - execution time
  - model breakdown
  - node breakdown
  - average cost per call
- **Key expressions or variables used:**  
  Builds:
  - `details`
  - `summary`
  - `modelBreakdown`
  - `nodeBreakdown`
- **Input and output connections:**  
  Receives from **Model Prices**; final output node of the workflow.
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases / failures:**  
  - If no items arrive, returns `{ error: 'No data to summarize' }`
  - If upstream merge omitted some metadata, summary still works because it mainly relies on per-call fields
  - Rounding is applied to cost totals only, not token counts
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When Called By Another Workflow | n8n-nodes-base.executeWorkflowTrigger | Sub-workflow entry point receiving upstream payload |  | Extract Execution ID | ##  Installation Steps<br>1. Go to **Settings → n8n API** and create an API key<br>2. Add it as credential for the **Get Execution Data** node<br>3. Review model mappings in **Standardize Names** node<br>4. Review pricing in **Model Prices** node<br><br>##  To Monitor a Workflow<br>1. Add **Execute Workflow** node at the end of your target workflow<br>2. Select this monitoring workflow<br>3. **Turn OFF** "Wait For Sub-Workflow Completion"<br>4. Pass `{ "executionId": "{{ $execution.id }}" }` as input<br><br>## ⚠️ Prerequisites<br>Enable **"Return Intermediate Steps"** in your AI Agent settings for best results. |
| Test with Execution ID | n8n-nodes-base.code | Disabled manual test input for a pasted execution ID |  | Extract Execution ID | ##  Installation Steps<br>1. Go to **Settings → n8n API** and create an API key<br>2. Add it as credential for the **Get Execution Data** node<br>3. Review model mappings in **Standardize Names** node<br>4. Review pricing in **Model Prices** node<br><br>##  To Monitor a Workflow<br>1. Add **Execute Workflow** node at the end of your target workflow<br>2. Select this monitoring workflow<br>3. **Turn OFF** "Wait For Sub-Workflow Completion"<br>4. Pass `{ "executionId": "{{ $execution.id }}" }` as input<br><br>## ⚠️ Prerequisites<br>Enable **"Return Intermediate Steps"** in your AI Agent settings for best results. |
| Extract Execution ID | n8n-nodes-base.set | Resolves execution ID from input, body, query, or current execution | When Called By Another Workflow; Test with Execution ID | Get Execution Data | ##  Installation Steps<br>1. Go to **Settings → n8n API** and create an API key<br>2. Add it as credential for the **Get Execution Data** node<br>3. Review model mappings in **Standardize Names** node<br>4. Review pricing in **Model Prices** node<br><br>##  To Monitor a Workflow<br>1. Add **Execute Workflow** node at the end of your target workflow<br>2. Select this monitoring workflow<br>3. **Turn OFF** "Wait For Sub-Workflow Completion"<br>4. Pass `{ "executionId": "{{ $execution.id }}" }` as input<br><br>## ⚠️ Prerequisites<br>Enable **"Return Intermediate Steps"** in your AI Agent settings for best results. |
| Get Execution Data | n8n-nodes-base.n8n | Fetches execution details from n8n API | Extract Execution ID | Extract Token Usage; Find Nodes with LLM Data | ##  Installation Steps<br>1. Go to **Settings → n8n API** and create an API key<br>2. Add it as credential for the **Get Execution Data** node<br>3. Review model mappings in **Standardize Names** node<br>4. Review pricing in **Model Prices** node<br><br>##  To Monitor a Workflow<br>1. Add **Execute Workflow** node at the end of your target workflow<br>2. Select this monitoring workflow<br>3. **Turn OFF** "Wait For Sub-Workflow Completion"<br>4. Pass `{ "executionId": "{{ $execution.id }}" }` as input<br><br>## ⚠️ Prerequisites<br>Enable **"Return Intermediate Steps"** in your AI Agent settings for best results. |
| Extract Token Usage | n8n-nodes-base.code | Recursively extracts token usage and LLM call details from execution data | Get Execution Data | Standardize Names | ## 🎯 Supported Providers<br>**OpenAI** · **Anthropic** · **Google** · **DeepSeek** · **Meta** · **Mistral** · **xAI** · **Cohere** · **Alibaba Qwen** · **Moonshot Kimi**<br><br>### 120+ Model Variations Mapped<br>Includes all versioned variants (e.g., gpt-4o-2024-08-06 → gpt-4o)<br><br>Prices sourced from official provider pages (March 2026) |
| Find Nodes with LLM Data | n8n-nodes-base.code | Extracts LLM-node metadata and dependency chain context | Get Execution Data | Merge | ## 🎯 Supported Providers<br>**OpenAI** · **Anthropic** · **Google** · **DeepSeek** · **Meta** · **Mistral** · **xAI** · **Cohere** · **Alibaba Qwen** · **Moonshot Kimi**<br><br>### 120+ Model Variations Mapped<br>Includes all versioned variants (e.g., gpt-4o-2024-08-06 → gpt-4o)<br><br>Prices sourced from official provider pages (March 2026) |
| Standardize Names | n8n-nodes-base.code | Maps raw model names to canonical model identifiers | Extract Token Usage | All Models Defined? | ## ⚙️ Defined by User<br>1. Define the model names in **standardize_names_dic**<br>2. If you want to use custom prices, update **MODEL_PRICES**<br>3. Set the costs of the model<br>4. Prices are per **1 million tokens**<br><br>### When You See Errors<br>If the workflow enters the error path, it means an **undefined model** was detected. Simply:<br>1. Add the model name to **standardize_names_dic**<br>2. Add its pricing to **MODEL_PRICES**<br>3. Re-run the workflow |
| All Models Defined? | n8n-nodes-base.if | Validates that each standardized model is known | Standardize Names | Merge; Stop and Error | ## ⚙️ Defined by User<br>1. Define the model names in **standardize_names_dic**<br>2. If you want to use custom prices, update **MODEL_PRICES**<br>3. Set the costs of the model<br>4. Prices are per **1 million tokens**<br><br>### When You See Errors<br>If the workflow enters the error path, it means an **undefined model** was detected. Simply:<br>1. Add the model name to **standardize_names_dic**<br>2. Add its pricing to **MODEL_PRICES**<br>3. Re-run the workflow |
| Stop and Error | n8n-nodes-base.code | Throws a blocking error for unknown models | All Models Defined? |  | ## ⚙️ Defined by User<br>1. Define the model names in **standardize_names_dic**<br>2. If you want to use custom prices, update **MODEL_PRICES**<br>3. Set the costs of the model<br>4. Prices are per **1 million tokens**<br><br>### When You See Errors<br>If the workflow enters the error path, it means an **undefined model** was detected. Simply:<br>1. Add the model name to **standardize_names_dic**<br>2. Add its pricing to **MODEL_PRICES**<br>3. Re-run the workflow |
| Merge | n8n-nodes-base.merge | Combines usage records with node metadata | All Models Defined?; Find Nodes with LLM Data | Model Prices |  |
| Model Prices | n8n-nodes-base.code | Applies per-model token pricing and computes costs | Merge | Generate Summary | ## 💡 You can do anything with this data!<br>- Store in a database for historical tracking<br>- Send to Teams as a cost alert<br>- Build dashboards with the summary data<br>- Set budget thresholds and trigger warnings<br>- Export to Google Sheets for reporting |
| Generate Summary | n8n-nodes-base.code | Builds final detailed output and aggregated usage summary | Model Prices |  | ## 📊 Output Data<br><br>### Per LLM Call<br>- Cost Breakdown (prompt, completion, total USD)<br>- Token Metrics (prompt, completion, total)<br>- Performance (execution time, finish reason)<br>- Content Preview (first 100 chars I/O)<br>- Model Parameters (temp, max tokens, timeout)<br>- Execution Context (workflow, node, status)<br>- Flow Tracking (previous nodes chain)<br><br>### Summary Statistics<br>- Total executions and costs<br>- Breakdown by model type<br>- Breakdown by node<br>- Average cost per call<br>- Total execution time |
| Sticky Note - Setup | n8n-nodes-base.stickyNote | Documentation note for setup and usage |  |  |  |
| Sticky Note - Output | n8n-nodes-base.stickyNote | Documentation note describing final output |  |  |  |
| Sticky Note - User Config | n8n-nodes-base.stickyNote | Documentation note for custom model/pricing maintenance |  |  |  |
| Sticky Note - Providers | n8n-nodes-base.stickyNote | Documentation note describing supported providers |  |  |  |
| Sticky Note - Next Steps | n8n-nodes-base.stickyNote | Documentation note suggesting downstream uses |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `LLM Cost Monitor - AI Usage Tracker`.
   - Keep the workflow active if you want it callable immediately.
   - In workflow settings:
     - `executionOrder`: `v1`
     - `binaryMode`: `separate`
     - `callerPolicy`: `workflowsFromSameOwner`

2. **Add the sub-workflow trigger**
   - Add node: **When Called By Another Workflow**
   - Type: `Execute Workflow Trigger`
   - Set **Input Source** to `Passthrough`
   - This is the main entry point for production use

3. **Add an optional manual test node**
   - Add node: **Code**
   - Name it `Test with Execution ID`
   - Disable the node
   - Paste this logic conceptually:
     - define a constant `testExecutionId`
     - return one item containing `{ executionId: testExecutionId }`
   - Use it only for manual debugging

4. **Add a Set node to normalize the execution ID**
   - Add node: **Set**
   - Name it `Extract Execution ID`
   - Create one field:
     - `executionId` as string
   - Use this expression:
     - `{{ $json.executionId || $json.body?.executionId || $json.query?.executionId || $execution.id }}`
   - This lets the workflow accept different incoming payload shapes

5. **Connect the entry nodes**
   - Connect `When Called By Another Workflow` → `Extract Execution ID`
   - Connect `Test with Execution ID` → `Extract Execution ID`

6. **Create the n8n API credential**
   - In n8n, go to **Settings → n8n API**
   - Generate an API key
   - Create a credential of type **n8n API**
   - Point it to the correct n8n instance URL
   - Ensure the credential has access to execution data

7. **Add the n8n node for execution retrieval**
   - Add node: **n8n**
   - Name it `Get Execution Data`
   - Attach the **n8n API** credential created above
   - Configure it to retrieve execution information for the provided execution ID
   - If reproducing exactly from template behavior, use the same credential type:
     - `n8nApi`
   - Connect:
     - `Extract Execution ID` → `Get Execution Data`

8. **Add the token extraction code node**
   - Add node: **Code**
   - Name it `Extract Token Usage`
   - Paste code that:
     - reads the execution payload
     - gets workflow and execution metadata
     - traverses `data.resultData.runData`
     - recursively searches for `tokenUsage` or `usage`
     - extracts:
       - prompt tokens
       - completion tokens
       - total tokens
       - model
       - finish reason
       - execution time
       - start time
       - output preview
       - input preview
       - temperature
       - max tokens
       - timeout
       - retry count
       - path
       - workflow metadata
     - returns one item per LLM call
     - returns an error item if no LLM calls are found
   - Connect:
     - `Get Execution Data` → `Extract Token Usage`

9. **Add the node-discovery code node**
   - Add node: **Code**
   - Name it `Find Nodes with LLM Data`
   - Paste code that:
     - reads workflow nodes and connections
     - reads run data
     - identifies likely LLM nodes from a hardcoded list plus type heuristics
     - computes previous-node chains recursively
     - returns metadata items containing:
       - nodeName
       - nodeType
       - isLLMNode
       - executionTime
       - startTime
       - previousNodes
       - position
   - Connect:
     - `Get Execution Data` → `Find Nodes with LLM Data`

10. **Add the model normalization node**
    - Add node: **Code**
    - Name it `Standardize Names`
    - Paste code that:
      - loops over all items
      - passes through the “no LLM calls” error item unchanged
      - converts raw model names to lowercase
      - looks them up in a large dictionary `standardize_names_dic`
      - tries partial matching if exact lookup fails
      - outputs:
        - `standardizedModel`
        - `modelKnown`
        - `rawModel`
    - Populate the dictionary with the models you need
    - Include the providers shown in the source workflow if you want equivalent coverage
    - Connect:
      - `Extract Token Usage` → `Standardize Names`

11. **Add the validation IF node**
    - Add node: **If**
    - Name it `All Models Defined?`
    - Configure one boolean condition:
      - left value: `{{ $json.modelKnown }}`
      - operation: equals
      - right value: `true`
    - Connect:
      - `Standardize Names` → `All Models Defined?`

12. **Add the strict error node**
    - Add node: **Code**
    - Name it `Stop and Error`
    - Paste code that:
      - filters unknown models
      - deduplicates them
      - throws an Error with instructions to update:
        - `standardize_names_dic`
        - `MODEL_PRICES`
    - Connect:
      - false output of `All Models Defined?` → `Stop and Error`

13. **Add the merge node**
    - Add node: **Merge**
    - Name it `Merge`
    - Set:
      - **Mode**: `Combine`
      - **Join mode**: `Enrich Input 1`
    - Connect:
      - true output of `All Models Defined?` → `Merge` input 1
      - `Find Nodes with LLM Data` → `Merge` input 2

14. **Add the pricing node**
    - Add node: **Code**
    - Name it `Model Prices`
    - Paste code that:
      - defines `MODEL_PRICES`
      - expects prices in USD per **1 million tokens**
      - reads `standardizedModel`
      - computes:
        - `promptCost`
        - `completionCost`
        - `totalCost`
        - `inputPricePerM`
        - `outputPricePerM`
        - `pricingFound`
      - rounds costs to 6 decimals
      - applies fallback pricing if a model has no entry
    - Connect:
      - `Merge` → `Model Prices`

15. **Add the summary node**
    - Add node: **Code**
    - Name it `Generate Summary`
    - Paste code that:
      - gathers all priced call records
      - computes totals:
        - total cost
        - total prompt tokens
        - total completion tokens
        - total tokens
        - total execution time
      - computes:
        - model breakdown
        - node breakdown
        - average cost per call
      - returns a single item:
        - `{ summary, details }`
    - Connect:
      - `Model Prices` → `Generate Summary`

16. **Add documentation sticky notes**
    - Add five Sticky Note nodes with these themes:
      - setup/install steps
      - output data description
      - user-configurable model/pricing instructions
      - supported providers
      - downstream ideas / next steps
    - These do not affect execution but are part of the original design

17. **Activate the workflow**
    - Save and activate it
    - Ensure the sub-workflow trigger is available to other workflows

18. **Configure the parent workflow to call this monitor**
    - In the target workflow you want to monitor, add an **Execute Workflow** node near the end
    - Select this monitoring workflow
    - Turn **OFF** `Wait For Sub-Workflow Completion`
    - Pass input data like:
      - `{ "executionId": "{{ $execution.id }}" }`
    - This sends the completed execution ID to the monitoring workflow

19. **Recommended AI workflow prerequisite**
    - In workflows using AI Agent / LangChain-style nodes, enable:
      - **Return Intermediate Steps**
    - This improves the chances that token usage appears in the execution payload and can be extracted

20. **Maintain the dictionaries over time**
    - Update `standardize_names_dic` whenever a provider introduces a new model name variant
    - Update `MODEL_PRICES` whenever provider pricing changes
    - Keep both dictionaries aligned so validation and pricing remain consistent

### Sub-workflow setup expectations
- **Input expected by this workflow:**
  - preferably `{ "executionId": "<target execution id>" }`
- **Output produced by this workflow:**
  - one final item containing:
    - `summary`
    - `details`
- **Behavior on unknown model:**
  - workflow throws an error and stops
- **Behavior on no LLM usage found:**
  - extraction returns an error item; depending on branch behavior, you may want to customize handling if you prefer graceful completion

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Go to **Settings → n8n API** and create an API key | Setup guidance for the n8n API credential |
| Add **Execute Workflow** at the end of the target workflow and pass `{ "executionId": "{{ $execution.id }}" }` | Integration pattern for monitored workflows |
| Turn OFF **Wait For Sub-Workflow Completion** | Recommended parent-workflow setting |
| Enable **Return Intermediate Steps** in AI Agent settings for best results | Improves extraction coverage |
| Supported providers include **OpenAI, Anthropic, Google, DeepSeek, Meta, Mistral, xAI, Cohere, Alibaba Qwen, Moonshot Kimi** | Coverage note |
| Prices are defined per **1 million tokens** | Pricing logic note |
| If the workflow errors on unknown models, update both **standardize_names_dic** and **MODEL_PRICES** | Maintenance guidance |
| You can store the results in a database, send alerts to Teams, build dashboards, apply budget thresholds, or export to Google Sheets | Downstream usage ideas |

## Additional implementation observations
- The workflow has **two entry paths**:
  - production: `When Called By Another Workflow`
  - manual testing: `Test with Execution ID` (disabled)
- There are **no external sub-workflows invoked by this workflow**
- The workflow is itself intended to be used **as a sub-workflow**
- The strongest operational dependency is the ability to retrieve detailed execution data from n8n, including AI intermediate results