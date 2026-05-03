Track AI Agent token usage and estimate costs in Google Sheets

https://n8nworkflows.xyz/workflows/track-ai-agent-token-usage-and-estimate-costs-in-google-sheets-5541


# Track AI Agent token usage and estimate costs in Google Sheets

---

### 1. Workflow Overview

This workflow tracks AI Agent token usage and estimates costs by extracting token usage data from AI language model executions and recording it into a Google Sheets spreadsheet. It is designed for scenarios where multiple AI language models or providers are used, and you want to monitor and estimate the associated token consumption costs centrally.

The workflow logically divides into these blocks:

- **1.1 Trigger and AI Agent Execution:** Initiates the workflow manually and runs an example AI Agent which calls a "think" tool twice.

- **1.2 Sub-Workflow Invocation and Execution Data Retrieval:** After AI Agent completion, calls a sub-workflow asynchronously to retrieve execution metadata, specifically token usage.

- **1.3 Token Usage Extraction and Aggregation:** Processes the execution metadata to extract detailed token usage per AI model, then aggregates totals by model.

- **1.4 Recording to Google Sheets:** Appends the aggregated token usage and estimated cost data into a specified Google Sheets file for tracking.

- **1.5 Supporting Notes and Credentials:** Contains auxiliary nodes with instructions, limitations, author info, and credentials configuration for Google Sheets and AI providers.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and AI Agent Execution

**Overview:**  
This block initiates the workflow manually and runs an example AI Agent node configured to demonstrate token usage data collection. It calls an internal "think" tool twice as part of the agent’s logic.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Think  
- AI Agent  
- Sticky Note3 (comment)

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual execution of the workflow.  
  - *Config:* No specific parameters; triggers workflow when user clicks "Test workflow."  
  - *Connections:* Outputs to AI Agent node.  
  - *Failures:* None typical; manual trigger.

- **Think**  
  - *Type:* LangChain Tool (Think)  
  - *Role:* Represents an internal "think" tool called by the AI Agent for intermediate processing.  
  - *Config:* Default, no parameters; responds to AI Agent’s tool calls.  
  - *Connections:* Invoked implicitly by AI Agent (node connection shown as ai_tool).  
  - *Failures:* Possible failures if tool logic fails or response is invalid.

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Runs an artificial intelligence agent that executes the prompt: "Help me test something and output the text 'This is a test workflow' after calling the think tool twice."  
  - *Config:* Text prompt defined inline, "define" prompt type, no extra options.  
  - *Connections:* Receives from manual trigger; outputs to "Call sub-workflow".  
  - *Failures:* API errors, timeouts, or prompt processing issues.  
  - *Notes:* This is an example AI Agent to demonstrate token tracking; user can replace with their own agents.  
  - *Sticky Note3:* Explains this node is an example and how it relates to subworkflow calls.

---

#### 2.2 Sub-Workflow Invocation and Execution Data Retrieval

**Overview:**  
After the AI Agent completes, this block asynchronously calls a sub-workflow that collects workflow execution metadata, including token usage data. It then retrieves and prepares this data for processing.

**Nodes Involved:**  
- Call sub-workflow  
- When Executed by Another Workflow  
- Get execution data  
- Extract token usage data  
- Sticky Note (positioned near Call sub-workflow)  
- Sticky Note2 (near When Executed by Another Workflow)

**Node Details:**  

- **Call sub-workflow**  
  - *Type:* Execute Workflow  
  - *Role:* Calls the same workflow as a sub-workflow asynchronously (note: waitForSubWorkflow option disabled).  
  - *Config:*  
    - Mode: "each" (process each item separately).  
    - Wait for Sub-Workflow Completion: disabled (to reduce data overload).  
    - Workflow ID: current workflow (`{{ $workflow.id }}`) — calls itself as sub-workflow.  
    - Inputs: passes current execution ID as `execution_id`.  
  - *Connections:* Receives from AI Agent; no output connections (end of main path).  
  - *Failures:* Reference errors if subworkflow ID changes or data mapping fails.  
  - *Sticky Note (near node):* Explains importance of disabling wait for sub-workflow completion to avoid messy data and that the subworkflow ID can be changed if using separate files.

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Trigger node for the sub-workflow, activated by the parent workflow call.  
  - *Config:* Expects input parameter `execution_id` from calling workflow.  
  - *Connections:* Outputs to Get execution data node.  
  - *Failures:* Missing `execution_id` parameter or trigger misconfiguration.

- **Get execution data**  
  - *Type:* n8n API Node (execution resource)  
  - *Role:* Fetches workflow execution metadata using `execution_id`.  
  - *Config:*  
    - Operation: "get" execution.  
    - Execution ID: from input parameter `execution_id`.  
    - Requires n8n API credentials.  
  - *Connections:* Outputs to Extract token usage data node.  
  - *Failures:* API authentication errors, invalid execution ID, or timeout.

- **Extract token usage data**  
  - *Type:* Set  
  - *Role:* Extracts token usage details from the execution metadata using JMESPath queries.  
  - *Config:*  
    - Assignments include `execution_id` and `tokenUsage`.  
    - Token usage extracted by querying nested JSON fields for each AI language model call: model name and token usage/prompt tokens/completion tokens.  
  - *Connections:* Outputs to Split Out node.  
  - *Failures:* JMESPath expression failure if metadata structure changes or token usage fields missing.

- **Sticky Note2**  
  - *Content:* Indicates that after the main workflow calls the subworkflow, total tokens will appear in the Executions spreadsheet. Provides user guidance.

---

#### 2.3 Token Usage Extraction and Aggregation

**Overview:**  
This block processes the extracted token usage data, splits the array into individual records, and sums token counts grouped by AI model and execution identifiers.

**Nodes Involved:**  
- Split Out  
- Sum Token Totals - aggregate by model

**Node Details:**  

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the `tokenUsage` array into individual items for aggregation.  
  - *Config:* Field to split out: `tokenUsage`. Includes all other fields.  
  - *Connections:* Outputs to Sum Token Totals - aggregate by model node.  
  - *Failures:* If `tokenUsage` is empty or malformed, split produces no output or errors.

- **Sum Token Totals - aggregate by model**  
  - *Type:* Summarize  
  - *Role:* Aggregates and sums token counts by model and execution identifiers.  
  - *Config:*  
    - Split by fields: `id, name, tokenUsage.model, execution_id`.  
    - Sum fields: `tokenUsage.tokenUsage.promptTokens` and `tokenUsage.tokenUsage.completionTokens`.  
  - *Connections:* Outputs to Record token usage node.  
  - *Failures:* Missing or inconsistent field names in input data can cause aggregation failures.

---

#### 2.4 Recording to Google Sheets

**Overview:**  
Aggregated token usage is appended as a new row into a dedicated Google Sheets document designed to calculate cost estimates using formulas.

**Nodes Involved:**  
- Record token usage  
- Sticky Note5 (instructions on using the Sheets template)  
- Sticky Note1 (limitations and caveats)

**Node Details:**  

- **Record token usage**  
  - *Type:* Google Sheets  
  - *Role:* Appends the token usage and metadata as a new row to the "Executions" sheet.  
  - *Config:*  
    - Operation: Append row.  
    - Document ID: Points to a shared Google Sheets template file.  
    - Sheet Name: `gid=0` (Executions sheet).  
    - Columns mapped: `llm_model`, `timestamp` (current time), `workflow_id`, `execution_id`, `input tokens` (prompt tokens sum), `workflow_name`, and `completion tokens` (completion tokens sum).  
    - Credentials: Google Sheets OAuth2 credentials required.  
  - *Connections:* Receives from Summarize node.  
  - *Failures:* Authentication errors, spreadsheet access denial, or schema mismatches.

- **Sticky Note5**  
  - *Content:* Instructions to make a copy of the shared Google Sheet template file, describing the two sheets ("Executions" and "LLM Pricing") and warning to avoid modifying green columns with formulas.

- **Sticky Note1**  
  - *Content:* Lists limitations of the workflow:  
    1. Does not account for prompt caching, so cost estimates may be higher than actual.  
    2. Not tested with audio or video inputs.  
    3. The cost is only an estimate, affected by batch requests, caching, or provider-specific cost reductions.

---

#### 2.5 Supporting Nodes and Credentials

**Overview:**  
Additional nodes provide language model options, credential setups, and contextual notes for users. They do not affect main workflow logic but are important for configuration and understanding.

**Nodes Involved:**  
- Gemini (Google Gemini LLM node)  
- OpenAI (OpenAI LLM node)  
- Anthropic (Anthropic LLM node)  
- Sticky Note4 (LLM pricing resources)  
- Sticky Note6 (community support info)  
- Sticky Note7 (overall workflow description)  
- Sticky Note8 (compatibility note for "Extract token usage data")  
- Sticky Note10 (author info)  
- Sticky Note16 (Scrapes Academy promotion)

**Node Details:**  

- **Gemini, OpenAI, Anthropic**  
  - *Type:* LangChain LLM nodes for different AI providers.  
  - *Role:* Examples of supported LLM providers; no direct connections in main workflow.  
  - *Config:* Model names and provider credentials set.  
  - *Failures:* Credential errors or API limits.

- **Sticky Notes**  
  - Provide instructions, links, author credits, help resources, and pricing references.  
  - Sticky Note4 includes multiple external links to LLM pricing websites for user reference.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                               | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                         |
|-------------------------------|----------------------------------|-----------------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                   | Workflow entry point                           |                             | AI Agent                    |                                                                                                                     |
| Think                         | LangChain Tool (Think)            | AI Agent internal tool called twice           | AI Agent (ai_tool)           | AI Agent                    |                                                                                                                     |
| AI Agent                      | LangChain Agent                  | Runs example AI prompt and calls subworkflow  | When clicking ‘Test workflow’| Call sub-workflow           | This is an example AI Agent. Use this only to understand how to call the subworkflow and obtain token amount.       |
| Call sub-workflow             | Execute Workflow                 | Calls subworkflow asynchronously after AI Agent | AI Agent                    |                             | Wait for the workflow to finish before calling the subworkflow. Disable `Wait For Sub-Workflow Completion`.          |
| When Executed by Another Workflow | Execute Workflow Trigger        | Triggers subworkflow with execution ID        |                             | Get execution data          | After main workflow calls subworkflow, total tokens appear in the Executions spreadsheet.                            |
| Get execution data            | n8n API (execution resource)      | Retrieves execution metadata by execution ID  | When Executed by Another Workflow | Extract token usage data    |                                                                                                                     |
| Extract token usage data      | Set                             | Extracts token usage and model details         | Get execution data           | Split Out                   | Works well with all the providers above. If using others, may need to adjust this node.                             |
| Split Out                    | Split Out                        | Splits token usage array into individual items | Extract token usage data     | Sum Token Totals - aggregate by model |                                                                                                                     |
| Sum Token Totals - aggregate by model | Summarize                     | Aggregates and sums tokens by model and execution | Split Out                   | Record token usage          |                                                                                                                     |
| Record token usage           | Google Sheets                   | Appends aggregated token usage to spreadsheet | Sum Token Totals - aggregate by model |                             | Make a copy of this Sheets file. Avoid changing green columns with formulas.                                        |
| Gemini                      | LangChain LLM (Google Gemini)    | Example of alternative AI provider             |                             |                             |                                                                                                                     |
| OpenAI                      | LangChain LLM (OpenAI)            | Example AI provider                             |                             |                             |                                                                                                                     |
| Anthropic                   | LangChain LLM (Anthropic)         | Example AI provider                             |                             |                             |                                                                                                                     |
| Sticky Note (near Call sub-workflow) | Sticky Note                    | Instruction about subworkflow call and wait settings |                             |                             | Wait for the workflow to finish before calling the subworkflow. Disable `Wait For Sub-Workflow Completion`.          |
| Sticky Note1                | Sticky Note                      | Limitations of workflow and cost estimates     |                             |                             | Limitations: no prompt caching, untested with audio/video, cost is estimate only.                                   |
| Sticky Note2                | Sticky Note                      | Instruction about token totals visibility      |                             |                             | After the main workflow calls the subworkflow, total tokens appear in the Executions spreadsheet.                   |
| Sticky Note3                | Sticky Note                      | Notes AI Agent example usage                    |                             |                             | This is an example AI Agent. Use to understand subworkflow and token amount retrieval.                              |
| Sticky Note4                | Sticky Note                      | LLM Pricing resources and links                 |                             |                             | Where to find LLM pricing? Links to multiple LLM pricing websites.                                                  |
| Sticky Note6                | Sticky Note                      | Community support and help links                 |                             |                             | Need help? Links to n8n community forums and Scrapes Academy community.                                            |
| Sticky Note7                | Sticky Note                      | Workflow description and requirements           |                             |                             | Describes workflow purpose, how it works, usage, and requirements.                                                 |
| Sticky Note8                | Sticky Note                      | Compatibility note about token extraction       |                             |                             | Works well with listed providers; may need adjustment for others.                                                  |
| Sticky Note10               | Sticky Note                      | Author information and contact details          |                             |                             | Author: Solomon, AI & Automation Educator. Includes social links and other templates.                              |
| Sticky Note16               | Sticky Note                      | Scrapes Academy promotion                        |                             |                             | Promotion for advanced n8n skills learning and earning.                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create LangChain Think Tool Node**  
   - Name: `Think`  
   - Type: LangChain Tool (Think)  
   - No parameters; default config.

3. **Create LangChain Agent Node**  
   - Name: `AI Agent`  
   - Type: LangChain Agent  
   - Parameters:  
     - Text prompt: "Help me test something and output the text 'This is a test workflow' after calling the think tool twice."  
     - Prompt type: define  
     - No additional options.  
   - Connect output of Manual Trigger to input of AI Agent.  
   - Connect AI Agent's `ai_tool` output to `Think` node (implicit linking inside LangChain nodes).  
   - Configure API credentials for your preferred AI provider if needed.

4. **Create Execute Workflow Node (Sub-Workflow Caller)**  
   - Name: `Call sub-workflow`  
   - Type: Execute Workflow  
   - Parameters:  
     - Mode: each  
     - Wait for Sub-Workflow Completion: **disable** (unchecked)  
     - Workflow ID: Use current workflow ID expression `{{ $workflow.id }}` or set to the subworkflow's ID if separated.  
     - Workflow Inputs: define input `execution_id` mapped to `{{$execution.id}}`.  
   - Connect AI Agent output to this node.

5. **Create Execute Workflow Trigger Node (Sub-Workflow Entry Point)**  
   - Name: `When Executed by Another Workflow`  
   - Type: Execute Workflow Trigger  
   - Parameters: define input parameter `execution_id`.

6. **Create n8n API Node to Get Execution Data**  
   - Name: `Get execution data`  
   - Type: n8n API node, resource: execution, operation: get  
   - Parameters: set `executionId` to `{{ $json.execution_id }}` from trigger input.  
   - Configure n8n API credentials.  
   - Connect output of Execute Workflow Trigger to this node.

7. **Create Set Node to Extract Token Usage Data**  
   - Name: `Extract token usage data`  
   - Type: Set  
   - Parameters:  
     - Assign `execution_id` from `{{ $('When Executed by Another Workflow').item.json.execution_id }}`  
     - Assign `tokenUsage` using JMESPath expression:  
       ```
       $jmespath(
         $json,
         "data.resultData.runData.*[] | [?data.ai_languageModel] | [].{model: data.ai_languageModel[0][0].json.response.generations[0][0].generationInfo.model_name || inputOverride.ai_languageModel[0][0].json.options.model_name || inputOverride.ai_languageModel[0][0].json.options.model, tokenUsage: data.ai_languageModel[0][0].json.tokenUsage || data.ai_languageModel[0][0].json.tokenUsageEstimate}"
       )
       ```  
     - Include fields: workflowData.id, workflowData.name, and others as needed.  
   - Connect output of Get execution data to this node.

8. **Create Split Out Node**  
   - Name: `Split Out`  
   - Type: Split Out  
   - Parameters:  
     - Field to split out: `tokenUsage`  
     - Include all other fields.  
   - Connect output of Extract token usage data to this node.

9. **Create Summarize Node to Sum Tokens**  
   - Name: `Sum Token Totals - aggregate by model`  
   - Type: Summarize  
   - Parameters:  
     - Fields to split by: `id, name, tokenUsage.model, execution_id`  
     - Fields to summarize (sum):  
       - `tokenUsage.tokenUsage.promptTokens`  
       - `tokenUsage.tokenUsage.completionTokens`  
   - Connect output of Split Out node to this node.

10. **Create Google Sheets Node to Record Token Usage**  
    - Name: `Record token usage`  
    - Type: Google Sheets  
    - Parameters:  
      - Operation: Append  
      - Document ID: Use the ID of your Google Sheets file (or use the shared template)  
      - Sheet Name: "Executions" (`gid=0`)  
      - Columns mapping:  
        - `llm_model`: `{{ $json.tokenUsage_model }}`  
        - `timestamp`: `{{ $now.format('yyyy-MM-dd HH:mm:ss') }}`  
        - `workflow_id`: `{{ $json.id }}`  
        - `execution_id`: `{{ $json.execution_id }}`  
        - `input tokens`: `{{ $json.sum_tokenUsage_tokenUsage_promptTokens }}`  
        - `workflow_name`: `{{ $json.name }}`  
        - `completion tokens`: `{{ $json.sum_tokenUsage_tokenUsage_completionTokens }}`  
      - Use Google Sheets OAuth2 credentials.  
    - Connect output of Summarize node to this node.

11. **(Optional) Add Auxiliary Nodes and Sticky Notes**  
    - Add LangChain LLM nodes for Gemini, OpenAI, Anthropic with appropriate credentials for testing or expansion.  
    - Add sticky notes containing instructions, limitations, pricing links, author info, and community resources for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Make a copy of the Google Sheets template file [**[TEMPLATE] Calculate LLM Token Usage**](https://docs.google.com/spreadsheets/d/1c9CeePI6ebNnIKogyJKHUpDWT6UEowpH9OwVtViadyE/edit?usp=sharing). The file contains two sheets: "Executions" for token data and "LLM Pricing" for cost estimates. Avoid editing green formula columns. | Google Sheets template for tracking token usage and cost calculations.                                                                                       |
| Limitations: 1) Does not account for prompt caching so cost estimates may be higher than actual. 2) Not tested with audio or video inputs. 3) Cost is an estimate; batch requests and caching may affect real costs.                                                                                                     | Sticky note within the workflow describing constraints and expectations when using this token tracking system.                                                |
| Where to find LLM pricing? Useful websites: [llm-price.com](https://llm-price.com), [llm-prices.com](https://llm-prices.com), [llmprices.dev](https://llmprices.dev/), [LLM Price Check](https://llmpricecheck.com/), [OpenRouter Models](https://openrouter.ai/models)                                                   | Useful external resources for up-to-date pricing on language model usage.                                                                                     |
| Need help? Visit the n8n community forums: https://community.n8n.io/c/questions/ or join the [Scrapes Academy](https://www.skool.com/scrapes/about?ref=21f10ad99f4d46ba9b8aaea8c9f58c34) community for advanced learning and support.                                                                                         | Support and learning communities for n8n users.                                                                                                              |
| Author: Solomon, AI & Automation Educator at Scrapes Academy. Contact via email automations.solomon@gmail.com, Telegram https://t.me/salomaoguilherme, LinkedIn https://www.linkedin.com/in/guisalomao/. More templates: https://n8n.io/creators/solomon/                                                                | Author and project credit with contact information and additional resources.                                                                                  |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. All processing strictly adheres to content policies and does not contain illegal, offensive, or protected material. All data handled is legal and public.

---