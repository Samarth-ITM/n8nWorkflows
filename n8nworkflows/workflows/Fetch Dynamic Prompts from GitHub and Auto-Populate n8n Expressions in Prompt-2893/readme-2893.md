Fetch Dynamic Prompts from GitHub and Auto-Populate n8n Expressions in Prompt

https://n8nworkflows.xyz/workflows/fetch-dynamic-prompts-from-github-and-auto-populate-n8n-expressions-in-prompt-2893


# Fetch Dynamic Prompts from GitHub and Auto-Populate n8n Expressions in Prompt

### 1. Workflow Overview

This workflow automates the process of fetching AI prompt templates stored in a GitHub repository, dynamically populating them with variables defined within n8n, validating the presence of all required variables, and then sending the fully formatted prompt to an AI agent for processing. It targets AI engineers, automation specialists, and content creators who need a scalable, error-resistant system for managing AI prompts without manual updates.

The workflow is logically divided into three main blocks:

- **1.1 Retrieve Prompt from GitHub:** Fetches the raw prompt file from a GitHub repository and extracts its textual content.
- **1.2 Extract & Validate Variables, Replace Placeholders:** Scans the prompt for variable placeholders, checks that all required variables are defined in the workflow, and replaces placeholders with actual values.
- **1.3 AI Processing & Output:** Sends the fully populated prompt to an AI agent for processing and outputs the AI-generated response.

Error handling is integrated to stop execution and report missing variables if any required placeholders are not supplied.

---

### 2. Block-by-Block Analysis

#### 2.1 Retrieve Prompt from GitHub

- **Overview:**  
  This block initiates the workflow, sets the repository and file path variables, fetches the prompt file from GitHub, and extracts the prompt text for further processing.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - setVars (Set Node)  
  - GitHub (GitHub API Node)  
  - Extract from File (Extract Text from File)  
  - SetPrompt (Set Node)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow manually for testing or execution.  
    - *Config:* No parameters; simply triggers downstream nodes.  
    - *Connections:* Outputs to `setVars`.  
    - *Edge Cases:* None; manual start.

  - **setVars**  
    - *Type:* Set Node  
    - *Role:* Defines all variables required for the workflow, including GitHub repo details and prompt variables.  
    - *Config:*  
      - `Account`: GitHub owner name (e.g., "TPGLLC-US")  
      - `repo`: Repository name (e.g., "PeresPrompts")  
      - `path`: Folder path in repo (e.g., "SEO/")  
      - `prompt`: Filename (e.g., "keyword_research.md")  
      - Variables for prompt replacement (e.g., `company`, `product`, `features`, `sector`)  
    - *Expressions:* Variables are referenced downstream via `$json.variableName`.  
    - *Connections:* Outputs to `GitHub`.  
    - *Edge Cases:* Variable names must exactly match placeholders in the prompt.

  - **GitHub**  
    - *Type:* GitHub Node (API)  
    - *Role:* Fetches the raw prompt file content from the specified GitHub repository and path.  
    - *Config:*  
      - Owner, repository, and file path are dynamically set from `setVars` node values.  
      - Operation: Get file content.  
      - Credentials: GitHub API credentials configured.  
    - *Connections:* Outputs to `Extract from File`.  
    - *Edge Cases:*  
      - Authentication errors if credentials invalid.  
      - File not found or incorrect path errors.  
      - Rate limiting by GitHub API.

  - **Extract from File**  
    - *Type:* Extract from File Node  
    - *Role:* Extracts plain text content from the fetched file data.  
    - *Config:* Operation set to extract text.  
    - *Connections:* Outputs to `SetPrompt`.  
    - *Edge Cases:* File content must be text; binary or unsupported formats may cause failure.

  - **SetPrompt**  
    - *Type:* Set Node  
    - *Role:* Stores the extracted prompt text in a JSON property `data` for downstream use.  
    - *Config:* Assigns `data` = extracted file content.  
    - *Connections:* Outputs to `Check All Prompt Vars Present`.  
    - *Edge Cases:* None significant.

---

#### 2.2 Extract & Validate Variables, Replace Placeholders

- **Overview:**  
  This block scans the prompt text for all variable placeholders in n8n expression format, verifies that all required variables are defined in the `setVars` node, and replaces placeholders with actual values. If any variables are missing, the workflow stops with an error.

- **Nodes Involved:**  
  - Check All Prompt Vars Present (Code Node)  
  - If (Conditional Node)  
  - replace variables (Code Node)  
  - Stop and Error (Stop Node)  
  - Set Completed Prompt (Set Node)

- **Node Details:**

  - **Check All Prompt Vars Present**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Parses the prompt text to extract all placeholders matching `{{ ... }}`, isolates variable names, and compares them against variables defined in `setVars`.  
    - *Key Logic:*  
      - Uses regex `/{{(.*?)}}/g` to find placeholders.  
      - Extracts variable names by trimming and taking the last segment after a dot (e.g., `{{ $json.company }}` → `company`).  
      - Compares extracted variable names with keys in `setVars`.  
      - Returns JSON with `success` boolean and array `missingKeys` of any missing variables.  
    - *Connections:* Outputs to `If` node.  
    - *Edge Cases:*  
      - If prompt contains malformed placeholders, extraction may fail or miss variables.  
      - Variables nested deeper than one dot are reduced to last segment, which may cause mismatches if naming is inconsistent.

  - **If**  
    - *Type:* Conditional Node  
    - *Role:* Routes workflow based on whether all required variables are present (`success` = true) or not.  
    - *Config:* Checks if `success` from previous node is true.  
    - *Connections:*  
      - True branch → `replace variables`  
      - False branch → `Stop and Error`  
    - *Edge Cases:* None.

  - **replace variables**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Replaces all placeholders in the prompt with actual variable values from `setVars`.  
    - *Key Logic:*  
      - Retrieves prompt text from `SetPrompt` node.  
      - Defines a variables object with keys and values from `setVars`.  
      - Uses regex `/{{(.*?)}}/g` to find placeholders and replaces them with corresponding variable values.  
      - Leaves placeholders unchanged if variable not found (though this should not occur due to prior validation).  
    - *Connections:* Outputs to `Set Completed Prompt`.  
    - *Edge Cases:*  
      - Hardcoded variables object in code includes some fixed values (e.g., `features: "Awesome Software"`), which may override `setVars` values unintentionally.  
      - If variable names in prompt and `setVars` differ, replacements may fail silently.

  - **Stop and Error**  
    - *Type:* Stop Node with Error  
    - *Role:* Stops workflow execution and returns an error message listing missing variables.  
    - *Config:* Error message dynamically includes missing variable names from `Check All Prompt Vars Present`.  
    - *Connections:* None (workflow ends here).  
    - *Edge Cases:* None.

  - **Set Completed Prompt**  
    - *Type:* Set Node  
    - *Role:* Stores the fully replaced prompt text in property `Prompt` for AI processing.  
    - *Config:* Assigns `Prompt` = output from `replace variables`.  
    - *Connections:* Outputs to `AI Agent`.  
    - *Edge Cases:* None.

---

#### 2.3 AI Processing & Output

- **Overview:**  
  This block sends the fully populated prompt to an AI agent (configured with Ollama Chat Model) and outputs the AI-generated response.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent Node)  
  - Prompt Output (Set Node)  
  - Ollama Chat Model (Language Model Node)

- **Node Details:**

  - **Set Completed Prompt** (from previous block)  
    - Outputs the final prompt to the AI Agent.

  - **AI Agent**  
    - *Type:* LangChain Agent Node  
    - *Role:* Sends the prompt text to the configured AI language model for processing.  
    - *Config:*  
      - Text input is set dynamically from `Prompt` property.  
      - Prompt type is set to "define" (custom prompt type).  
    - *Connections:* Outputs to `Prompt Output`.  
    - *Edge Cases:*  
      - AI model errors, timeouts, or API limits.  
      - Model-specific configuration may be required for other AI providers.

  - **Prompt Output**  
    - *Type:* Set Node  
    - *Role:* Stores the AI response in property `promptResponse` for downstream use or output.  
    - *Config:* Assigns `promptResponse` = AI Agent output `output` property.  
    - *Connections:* None (end of workflow).  
    - *Edge Cases:* None.

  - **Ollama Chat Model**  
    - *Type:* Language Model Node (Ollama)  
    - *Role:* Provides the AI model endpoint used by the AI Agent node.  
    - *Config:* Uses Ollama API credentials.  
    - *Connections:* Connected as AI language model resource for `AI Agent`.  
    - *Edge Cases:* Authentication or network errors with Ollama API.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                                  | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                      |
|---------------------------|--------------------------------|-------------------------------------------------|------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                 | Starts workflow manually                         |                              | setVars                         |                                                                                                 |
| setVars                   | Set Node                      | Defines GitHub repo info and prompt variables   | When clicking ‘Test workflow’ | GitHub                         | # Set The variables in your prompt here                                                         |
| GitHub                    | GitHub API Node               | Fetches prompt file from GitHub                  | setVars                      | Extract from File               | ## The repo is currently public for you to test with                                            |
| Extract from File         | Extract from File Node        | Extracts text content from GitHub file           | GitHub                       | SetPrompt                      |                                                                                                 |
| SetPrompt                 | Set Node                      | Stores extracted prompt text                      | Extract from File            | Check All Prompt Vars Present   |                                                                                                 |
| Check All Prompt Vars Present | Code Node (JS)               | Extracts variables from prompt and validates     | SetPrompt                   | If                            |                                                                                                 |
| If                        | Conditional Node              | Routes based on variable presence                 | Check All Prompt Vars Present | replace variables / Stop and Error |                                                                                                 |
| replace variables         | Code Node (JS)                | Replaces placeholders with actual variable values | If (true branch)             | Set Completed Prompt            | ## Replaces the values in the prompt with the variables in the 'setVars' Node                   |
| Stop and Error            | Stop Node with Error          | Stops workflow and reports missing variables      | If (false branch)            |                                 | ## If you're missing variables they will be listed here                                        |
| Set Completed Prompt      | Set Node                      | Stores fully replaced prompt                       | replace variables            | AI Agent                      |                                                                                                 |
| AI Agent                  | LangChain Agent Node          | Sends prompt to AI model and receives response    | Set Completed Prompt         | Prompt Output                  |                                                                                                 |
| Prompt Output             | Set Node                      | Stores AI response                                 | AI Agent                    |                                 |                                                                                                 |
| Ollama Chat Model         | Language Model Node (Ollama)  | Provides AI model endpoint                         |                             | AI Agent (as language model)   |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Create a Set Node for Variables**  
   - Name: `setVars`  
   - Add the following string fields with exact names and values:  
     - `Account`: GitHub owner name (e.g., "TPGLLC-US")  
     - `repo`: Repository name (e.g., "PeresPrompts")  
     - `path`: Folder path in repo (e.g., "SEO/")  
     - `prompt`: Filename (e.g., "keyword_research.md")  
     - `company`: e.g., "South Nassau Physical Therapy"  
     - `product`: e.g., "Manual Therapy"  
     - `features`: e.g., "pain relief"  
     - `sector`: e.g., "physical therapy"  
   - Connect output of Manual Trigger to this node.

3. **Create a GitHub Node**  
   - Name: `GitHub`  
   - Resource: File  
   - Operation: Get  
   - Owner: Expression `={{ $json.Account }}`  
   - Repository: Expression `={{ $json.repo }}`  
   - File Path: Expression `={{ $json.path }}{{ $json.prompt }}`  
   - Credentials: Configure GitHub API credentials.  
   - Connect output of `setVars` to this node.

4. **Create an Extract from File Node**  
   - Name: `Extract from File`  
   - Operation: Text extraction  
   - Connect output of `GitHub` to this node.

5. **Create a Set Node to Store Prompt**  
   - Name: `SetPrompt`  
   - Assign field `data` = `={{ $json.data }}` (the extracted text)  
   - Connect output of `Extract from File` to this node.

6. **Create a Code Node to Check Variables**  
   - Name: `Check All Prompt Vars Present`  
   - JavaScript code:  
     ```javascript
     const prompt = $json.data;
     const matches = [...prompt.matchAll(/{{(.*?)}}/g)];
     const uniqueVars = [...new Set(matches.map(m => m[1].trim().split('.').pop()))];
     const setNodeVariables = $node["setVars"].json || {};
     const missingKeys = uniqueVars.filter(v => !setNodeVariables.hasOwnProperty(v));
     return [{ success: missingKeys.length === 0, missingKeys }];
     ```
   - Connect output of `SetPrompt` to this node.

7. **Create an If Node**  
   - Name: `If`  
   - Condition: Check if `success` from previous node is true (`={{ $json.success }}`)  
   - Connect output of `Check All Prompt Vars Present` to this node.

8. **Create a Code Node to Replace Variables**  
   - Name: `replace variables`  
   - JavaScript code:  
     ```javascript
     const prompt = $node["SetPrompt"].json.data;
     const variables = $node["setVars"].json;
     const replaced = prompt.replace(/{{(.*?)}}/g, (match, key) => {
       const finalKey = key.trim().split('.').pop();
       return variables.hasOwnProperty(finalKey) ? variables[finalKey] : match;
     });
     return [{ prompt: replaced }];
     ```
   - Connect the `true` output of `If` to this node.

9. **Create a Stop and Error Node**  
   - Name: `Stop and Error`  
   - Error message: `=Missing Prompt Variables : {{ $json.missingKeys }}`  
   - Connect the `false` output of `If` to this node.

10. **Create a Set Node to Store Completed Prompt**  
    - Name: `Set Completed Prompt`  
    - Assign field `Prompt` = `={{ $json.prompt }}` (output from `replace variables`)  
    - Connect output of `replace variables` to this node.

11. **Create an Ollama Chat Model Node**  
    - Name: `Ollama Chat Model`  
    - Configure credentials with valid Ollama API account.  
    - No additional parameters needed.  

12. **Create a LangChain Agent Node**  
    - Name: `AI Agent`  
    - Text input: `={{ $json.Prompt }}`  
    - Prompt type: `define`  
    - Connect `Set Completed Prompt` output to this node.  
    - In the node settings, select the `Ollama Chat Model` as the language model resource.

13. **Create a Set Node for Prompt Output**  
    - Name: `Prompt Output`  
    - Assign field `promptResponse` = `={{ $json.output }}` (AI Agent output)  
    - Connect output of `AI Agent` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The prompt file in GitHub must contain variables in n8n expression format, e.g., `{{ $json.company }}`. | Ensures dynamic replacement works correctly.                                                       |
| The workflow currently uses the Ollama Chat Model but can be modified to use OpenAI, Claude, or other AI models. | Flexibility for AI provider integration.                                                           |
| Variables in the `setVars` node must exactly match the variable names used in the prompt placeholders. | Prevents missing variable errors.                                                                   |
| If any required variables are missing, the workflow stops and returns an error listing them.     | Prevents incomplete prompts from being sent to AI agents.                                          |
| The GitHub repository used in the example is public for testing purposes.                        | Users should configure their own private or public repos accordingly.                              |
| For dynamic variable sourcing, connect `setVars` to external data sources like Airtable or Google Sheets. | Enables CRM or database integration for variable values.                                           |
| Workflow is compatible with n8n Cloud, Self-Hosted, and Desktop versions.                        | Broad deployment compatibility.                                                                    |

---

This documentation provides a complete, detailed reference for understanding, reproducing, and customizing the workflow to dynamically fetch, validate, and process AI prompts stored in GitHub repositories using n8n.