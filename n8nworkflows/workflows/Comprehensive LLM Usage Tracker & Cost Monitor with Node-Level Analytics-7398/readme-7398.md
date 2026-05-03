Comprehensive LLM Usage Tracker & Cost Monitor with Node-Level Analytics

https://n8nworkflows.xyz/workflows/comprehensive-llm-usage-tracker---cost-monitor-with-node-level-analytics-7398


# Comprehensive LLM Usage Tracker & Cost Monitor with Node-Level Analytics

### 1. Workflow Overview

This workflow is designed as a **Comprehensive LLM Usage Tracker & Cost Monitor with Node-Level Analytics**. Its primary purpose is to track the usage of large language models (LLMs) across n8n workflow executions, extract detailed token usage data at the node level, standardize and validate model names, calculate associated costs based on predefined pricing, and produce comprehensive summaries for monitoring and cost management.

The workflow is well-suited for organizations or developers who run multiple LLM integrations in their n8n workflows and want to gain insights into usage patterns, token consumption, cost attribution per model and node, and detect any discrepancies in model naming or pricing definitions.

The workflow logic is grouped into the following functional blocks:

- **1.1 Input Reception and Execution Retrieval:** Triggered by execution ID input, fetches full execution details.
- **1.2 Model Extraction and Standardization:** Extracts all unique models used, then standardizes model names and defines pricing.
- **1.3 Validation of Model Definitions:** Checks that all detected models are correctly defined in standardization and pricing dictionaries, halting on errors.
- **1.4 LLM Usage Data Extraction:** Parses execution data to extract detailed LLM token usage and metadata per node.
- **1.5 Cost Calculation and Summary Analytics:** Calculates token costs per node and model, aggregates statistics, and prepares summary output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Execution Retrieval

**Overview:**  
This block receives an execution ID input (manually or via workflow trigger), retrieves the full workflow execution data from n8n, and forwards it for model extraction and usage analysis.

**Nodes Involved:**  
- When Exc. (Execute Workflow Trigger)  
- Test id (Manual Trigger)  
- Get an execution (n8n API Node)

**Node Details:**

- **When Exc.**  
  - Type: `Execute Workflow Trigger`  
  - Role: Starts workflow on external trigger with input parameter `execution_id` (number).  
  - Config: Accepts `execution_id` as input.  
  - Output: Passes execution ID to "Get an execution".  
  - Edge Cases: Invalid or missing execution ID may cause downstream failures.

- **Test id**  
  - Type: `Manual Trigger`  
  - Role: Allows manual triggering with preset test execution ID.  
  - Config: No parameters; uses pinned data with `execution_id=353`.  
  - Output: Passes execution ID to "Get an execution".  

- **Get an execution**  
  - Type: `n8n` (native node to fetch execution data)  
  - Role: Retrieves detailed workflow execution data from n8n.  
  - Config: Uses execution ID from input JSON (`{{$json.execution_id}}`). Requires n8n API credentials.  
  - Output: Emits full execution JSON for downstream processing.  
  - Failure Modes: API errors, invalid ID, or permission failures.

---

#### 2.2 Model Extraction and Standardization

**Overview:**  
Extracts all unique LLM model names used within the execution data, then maps them to standardized model names and defines a pricing dictionary for cost calculations.

**Nodes Involved:**  
- Extract all model names (Code)  
- Standardize names (Set)  
- model prices (Set)

**Node Details:**

- **Extract all model names**  
  - Type: `Code`  
  - Role: Parses execution data to find all unique model names used, scanning multiple possible fields where model names may appear.  
  - Config: JavaScript code iterates through runs, collecting unique strings representing model names.  
  - Output: Outputs object with `models_used` (array of model names), `model_count`, and original execution data.  
  - Edge Cases: Missing or malformed model fields, empty execution data.

- **Standardize names**  
  - Type: `Set`  
  - Role: Defines a dictionary mapping variant model names to standardized keys (e.g. `"gpt-4.1-mini": "gpt-4.1-mini"`).  
  - Config: JSON object with `standardize_names_dic` dictionary. Passes through `models_used`.  
  - Output: Adds standardization mapping for downstream validation.

- **model prices**  
  - Type: `Set`  
  - Role: Defines a pricing dictionary with input/output token prices per model (per million tokens).  
  - Config: JSON object with `model_price_dic` mapping model names to `{ input: price, output: price }`.  
  - Output: Provides pricing info for cost calculations.  
  - Notes: Prices are user-defined and must correspond to standardized model names.  
  - Sticky Note: Explains importance of defining model names and prices correctly.

---

#### 2.3 Validation of Model Definitions

**Overview:**  
Validates that all detected models are present in both the standardization and pricing dictionaries. If some models are missing, the workflow stops with an error reporting missing definitions.

**Nodes Involved:**  
- Check correctly defined (Code)  
- If not passed (If)  
- Stop and Error (Stop & Error)  
- Merge (Merge)

**Node Details:**

- **Check correctly defined**  
  - Type: `Code`  
  - Role: Compares models detected against the standardize names and pricing dictionaries, building error messages if mismatches found.  
  - Config: JS logic returns JSON with `passed` boolean, messages, and arrays of missing models.  
  - Output: Validation result object.

- **If not passed**  
  - Type: `If`  
  - Role: Checks if validation passed (`$json.passed === false`).  
  - True branch: Triggers error node.  
  - False branch: Continues workflow.

- **Stop and Error**  
  - Type: `Stop and Error`  
  - Role: Stops workflow execution with a formatted error message listing missing models in standardization and pricing dictionaries.  
  - Config: Uses expressions to display missing models.  
  - Notes: Prevents cost calculation on incomplete data.

- **Merge**  
  - Type: `Merge`  
  - Role: Combines validation success branch outputs with downstream LLM usage extraction outputs for cost calculation.  

---

#### 2.4 LLM Usage Data Extraction

**Overview:**  
Extracts detailed usage data per node from the execution, including token counts, model info, execution metadata, and intermediate content previews.

**Nodes Involved:**  
- Smart Extract LLM data (Code)

**Node Details:**

- **Smart Extract LLM data**  
  - Type: `Code`  
  - Role: Parses the execution's run data node by node, extracting promptTokens, completionTokens, model used, execution status, timing info, and sample prompt/response snippets.  
  - Config: Complex JS processing that handles multiple nested data structures and fallback locations for token usage and model parameters.  
  - Output: Outputs one item per LLM usage instance found, including metadata such as workflowName, node name, tokens, costs, and context chain.  
  - Edge Cases: Handles missing tokens, unknown models, empty runs, and no LLM usage scenarios by outputting summary items with zero tokens.

---

#### 2.5 Cost Calculation and Summary Analytics

**Overview:**  
Calculates token costs per usage item based on standardized model pricing, aggregates totals and statistics by model and node, and produces a final summary object.

**Nodes Involved:**  
- Calculate cost (Code)  
- Sticky Note6 (informational)

**Node Details:**

- **Calculate cost**  
  - Type: `Code`  
  - Role:  
    - Receives all LLM usage data and configuration dictionaries.  
    - Standardizes model names for each usage.  
    - Computes prompt and completion costs using prices per million tokens.  
    - Aggregates totals across all executions, grouping by model and node.  
    - Formats cost values to 8 decimal places.  
    - Appends a summary object as final output item.  
  - Output: Multiple items, each representing an LLM usage with cost details, plus one summary item.  
  - Edge Cases: No LLM usage data leads to a message output.  
  - Sticky Note6: Notes possibilities for downstream actions like sending cost notifications or triggering follow-up processes.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                                | Input Node(s)           | Output Node(s)            | Sticky Note                                            |
|-----------------------|----------------------------------|------------------------------------------------|-------------------------|---------------------------|--------------------------------------------------------|
| When Exc.             | Execute Workflow Trigger          | Workflow start with execution_id input         |                         | Get an execution          |                                                        |
| Test id               | Manual Trigger                   | Manual start with test execution_id             |                         | Get an execution          | "283\n353" (pinned test IDs)                           |
| Get an execution      | n8n API Node (execution retrieval) | Fetch full workflow execution details by ID    | When Exc., Test id       | Extract all model names, Smart Extract LLM data |                                                        |
| Extract all model names | Code                            | Extract unique LLM model names from execution  | Get an execution         | Standardize names         |                                                        |
| Standardize names     | Set                              | Define mappings for model name standardization | Extract all model names  | model prices              |                                                        |
| model prices          | Set                              | Define pricing per model (input/output tokens) | Standardize names        | Check correctly defined   | "Define the model name and standard name, even if you want to use same name. (Why? because we will use the standard name to find the cost of this model)\nModel prices are per million token" |
| Check correctly defined | Code                           | Validate all models are defined in dictionaries | model prices             | If not passed             |                                                        |
| If not passed         | If                               | Branch based on validation result               | Check correctly defined  | Stop and Error (true), Merge (false) |                                                        |
| Stop and Error        | Stop and Error                   | Stops and reports missing model definitions     | If not passed (true)     |                           | "In case you did something incorrectly, you can see what models you missed to add and define" |
| Smart Extract LLM data | Code                            | Extract detailed LLM usage data per node       | Get an execution         | Merge                     |                                                        |
| Merge                 | Merge                            | Combines LLM usage and validation outputs       | If not passed (false), Smart Extract LLM data | Calculate cost            |                                                        |
| Calculate cost        | Code                            | Calculate cost per usage and aggregate summary | Merge                    |                           | "You can do anything with this info:\n- Send a follow-up message with cost\n- Send another request to continue a process ..." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add an **Execute Workflow Trigger** node named `When Exc.`:
     - Add a workflow input named `execution_id` of type Number.
   - Add a **Manual Trigger** node named `Test id` (for manual testing).

2. **Retrieve Execution Data:**
   - Add an **n8n** node named `Get an execution`:
     - Set resource: `execution`
     - Operation: `get`
     - Set `executionId` parameter to `={{ $json.execution_id }}`
     - Configure credentials for n8n API access.
   - Connect outputs of `When Exc.` and `Test id` nodes to `Get an execution`.

3. **Extract All Model Names:**
   - Add a **Code** node named `Extract all model names`.
   - Paste the provided JavaScript code that scans execution data for unique model names.
   - Connect output of `Get an execution` to this node.

4. **Standardize Model Names:**
   - Add a **Set** node named `Standardize names`.
   - Configure JSON output with a dictionary `standardize_names_dic` mapping variant names to standard names, e.g.:
     ```json
     {
       "gpt-4.1-mini": "gpt-4.1-mini",
       "gpt-4": "gpt-4"
     }
     ```
   - Pass through `models_used` from previous node.
   - Connect output of `Extract all model names` to this node.

5. **Define Model Prices:**
   - Add a **Set** node named `model prices`.
   - Configure JSON output with a `model_price_dic` dictionary mapping models to input/output prices per million tokens, e.g.:
     ```json
     {
       "gpt-5": { "input": 1.25, "output": 10.0 },
       "gpt-4.1-mini": { "input": 0.4, "output": 1.6 },
       ...
     }
     ```
   - Ensure keys match standardized names.
   - Connect output of `Standardize names` to this node.

6. **Validate Model Definitions:**
   - Add a **Code** node named `Check correctly defined`.
   - Paste provided JavaScript that compares detected models against standardize names and prices dictionaries, returns pass/fail.
   - Connect output of `model prices` to this node.

7. **Branch on Validation:**
   - Add an **If** node named `If not passed`.
   - Condition: Expression checks if `{{$json.passed}}` is false.
   - Connect output of `Check correctly defined` to this node.

8. **Add Stop and Error Node:**
   - Add a **Stop and Error** node named `Stop and Error`.
   - Configure error message using expressions to display missing models.
   - Connect `If not passed` True output to this node.

9. **Extract Detailed LLM Usage:**
   - Add a **Code** node named `Smart Extract LLM data`.
   - Paste the provided JavaScript for detailed extraction of tokens, models, execution info, prompts, responses, and metadata.
   - Connect output of `Get an execution` also directly to this node.

10. **Merge Validated and Usage Data:**
    - Add a **Merge** node named `Merge`.
    - Connect `If not passed` False output and `Smart Extract LLM data` output to `Merge`.
    - Configure default merge mode (e.g., Append).

11. **Calculate Costs and Summary:**
    - Add a **Code** node named `Calculate cost`.
    - Paste provided JavaScript calculating costs using standardized names and prices, aggregates by model and node, formats results.
    - Connect output of `Merge` to this node.

12. **Credentials Setup:**
    - Ensure `Get an execution` node has configured n8n API credentials.
    - No other external credentials required.

13. **Sticky Notes:**
    - Add sticky notes as per original workflow to explain model price setup, testing instructions, and usage notes.

14. **Testing:**
    - Use `Test id` manual trigger with pinned execution ID (e.g., 353) or trigger `When Exc.` externally with execution ID.
    - Verify outputs at each stage, especially validation and cost calculation.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Model prices must be defined per million tokens for input and output separately.                               | Sticky note on "model prices" node.                                                                                 |
| Execution IDs can be found in the n8n UI under "Executions" tab, typically numeric like 323 or 353.            | Sticky note near triggers explaining how to find execution ID.                                                     |
| If models are missing from standardization or pricing dictionaries, the workflow stops with an error message.  | Sticky note near "Stop and Error" node to help users debug missing model definitions.                               |
| Final cost data can be used to trigger follow-up notifications or further automation steps.                    | Sticky note near "Calculate cost" node suggesting next steps with extracted cost data.                             |
| Workflow requires n8n API credentials with permissions to read executions.                                     | Credential requirement noted in "Get an execution" node configuration.                                             |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.