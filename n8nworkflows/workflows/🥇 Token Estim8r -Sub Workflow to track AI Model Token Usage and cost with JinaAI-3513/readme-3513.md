🥇 Token Estim8r -Sub Workflow to track AI Model Token Usage and cost with JinaAI

https://n8nworkflows.xyz/workflows/---token-estim8r--sub-workflow-to-track-ai-model-token-usage-and-cost-with-jinaai-3513


# 🥇 Token Estim8r -Sub Workflow to track AI Model Token Usage and cost with JinaAI

### 1. Workflow Overview

This workflow, titled **"🥇 Token Estim8r -Sub Workflow to track AI Model Token Usage and cost with JinaAI"**, is designed to estimate and log token usage and associated costs for AI models used within n8n workflows. It targets AI engineers, automation specialists, and business analysts who leverage token-based large language models (LLMs) such as OpenAI or Anthropic and want detailed visibility into token consumption and cost per workflow execution.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception**: Entry points to trigger the workflow manually or from another workflow.
- **1.2 Execution Data Retrieval**: Fetches detailed execution data of the workflow run to analyze.
- **1.3 AI Usage Data Extraction**: Processes execution data to extract token usage and model/tool information.
- **1.4 Data Preparation and Pricing Retrieval**: Sets up data for pricing, optionally fetches live AI pricing from Jina AI API.
- **1.5 Cost Calculation and Summary**: Calculates total cost based on tokens and pricing, prepares summary data.
- **1.6 Logging to Google Sheets**: Records the token usage and cost data into a Google Sheet for tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block provides two entry points to trigger the workflow: manually via a manual trigger node or programmatically via an execute workflow trigger node from another workflow.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - When Executed by Another Workflow (Execute Workflow Trigger)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the workflow for testing or ad-hoc runs.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connected to "n8n Get Execution Data" node.  
    - Edge Cases: None significant; manual trigger may be idle if not used.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Enables this workflow to be called as a sub-workflow from other workflows, passing execution context.  
    - Configuration: Default, listens for execution calls.  
    - Inputs: Triggered by external workflow execution.  
    - Outputs: Connected to "n8n Get Execution Data" node.  
    - Edge Cases: If called without proper execution context, downstream nodes may fail.

---

#### 1.2 Execution Data Retrieval

- **Overview:**  
  Retrieves detailed execution data of the workflow run identified by the execution ID, which is essential for analyzing token usage.

- **Nodes Involved:**  
  - n8n Get Execution Data

- **Node Details:**

  - **n8n Get Execution Data**  
    - Type: n8n Node (n8n core node for fetching execution data)  
    - Role: Fetches the full execution data of the workflow run using the execution ID passed from the trigger nodes.  
    - Configuration: Uses the execution ID input (e.g., `{{ $json["executionId"] }}` or passed from parent workflow).  
    - Inputs: From either manual trigger or execute workflow trigger.  
    - Outputs: Connected to "Get AI Usage Data" node.  
    - Edge Cases:  
      - Execution ID missing or invalid leads to failure or empty data.  
      - API rate limits or n8n internal errors may cause timeouts.

---

#### 1.3 AI Usage Data Extraction

- **Overview:**  
  Processes the retrieved execution data to extract relevant AI usage metrics such as prompt tokens, completion tokens, models used, and tools involved.

- **Nodes Involved:**  
  - Get AI Usage Data

- **Node Details:**

  - **Get AI Usage Data**  
    - Type: Code Node (JavaScript/TypeScript)  
    - Role: Parses the execution data JSON to identify AI model token usage and tools utilized during the workflow run.  
    - Configuration: Custom code that traverses execution data to sum tokens and extract model/tool metadata.  
    - Inputs: Execution data from "n8n Get Execution Data".  
    - Outputs: Structured AI usage data passed to "Set Ai_Run_Data".  
    - Key Expressions: Uses JSON parsing and aggregation logic inside code.  
    - Edge Cases:  
      - Unexpected execution data structure may cause parsing errors.  
      - Missing token data if upstream nodes do not log usage properly.

---

#### 1.4 Data Preparation and Pricing Retrieval

- **Overview:**  
  Prepares the AI run data for pricing calculation and optionally fetches live AI model pricing from Jina AI API, continuing gracefully on failure.

- **Nodes Involved:**  
  - Set Ai_Run_Data  
  - Get AI Pricing

- **Node Details:**

  - **Set Ai_Run_Data**  
    - Type: Set Node  
    - Role: Organizes and formats the AI usage data into variables for pricing calculation.  
    - Configuration: Sets key fields such as total tokens, prompt tokens, completion tokens, models used, and tools used.  
    - Inputs: From "Get AI Usage Data".  
    - Outputs: To "Get AI Pricing".  
    - Edge Cases: None significant; data integrity depends on prior node.

  - **Get AI Pricing**  
    - Type: HTTP Request  
    - Role: Retrieves live AI model pricing data from Jina AI API.  
    - Configuration:  
      - HTTP method: GET  
      - URL: Jina AI pricing endpoint (configured in parameters)  
      - Auth Header: Optional Jina API Auth Header (user-configurable)  
      - On Error: Continue workflow execution even if request fails.  
      - Retry on Fail: Enabled to handle transient network issues.  
    - Inputs: From "Set Ai_Run_Data".  
    - Outputs: To "Get Models Price and Add Summary".  
    - Edge Cases:  
      - Network failures or invalid API key cause fallback to static pricing.  
      - API response format changes may break parsing downstream.

---

#### 1.5 Cost Calculation and Summary

- **Overview:**  
  Calculates total cost based on token counts and pricing data, and prepares a summary record for logging.

- **Nodes Involved:**  
  - Get Models Price and Add Summary

- **Node Details:**

  - **Get Models Price and Add Summary**  
    - Type: Code Node  
    - Role: Combines token usage data with pricing (live or static) to compute total cost and formats a summary object.  
    - Configuration: Custom code that:  
      - Maps models to prices  
      - Calculates costs for prompt and completion tokens  
      - Aggregates total cost  
      - Prepares JSON array for logging  
    - Inputs: Pricing data from "Get AI Pricing" and AI run data.  
    - Outputs: To "Google Sheets".  
    - Edge Cases:  
      - Missing pricing data leads to fallback or zero cost.  
      - Unexpected model names or token counts may cause calculation errors.

---

#### 1.6 Logging to Google Sheets

- **Overview:**  
  Logs the token usage and cost summary data into a configured Google Sheet for persistent tracking and analysis.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets Node  
    - Role: Appends a new row with timestamp, token counts, models used, tools used, total cost, and JSON summary to a Google Sheet.  
    - Configuration:  
      - Operation: Append Row  
      - Sheet and Spreadsheet ID: User-configured to point to the prepared Google Sheet.  
      - Data fields mapped to columns: timestamp, Total Tokens, Prompt Tokens, Completion Tokens, Models Used, Tools Used, Total Cost, Json Array.  
    - Inputs: From "Get Models Price and Add Summary".  
    - Outputs: None (terminal node).  
    - Edge Cases:  
      - Authentication errors if Google credentials are invalid.  
      - Sheet not found or permission denied errors.  
      - Rate limits on Google Sheets API.

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                                  | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                       |
|-----------------------------|----------------------------|-------------------------------------------------|-------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger             | Manual entry point to start workflow             | None                          | n8n Get Execution Data         |                                                                                                 |
| When Executed by Another Workflow | Execute Workflow Trigger | Programmatic entry point from other workflows    | None                          | n8n Get Execution Data         |                                                                                                 |
| n8n Get Execution Data       | n8n core node              | Fetches execution data of workflow run            | When clicking ‘Test workflow’, When Executed by Another Workflow | Get AI Usage Data              |                                                                                                 |
| Get AI Usage Data            | Code Node                  | Extracts token usage and model/tool info          | n8n Get Execution Data         | Set Ai_Run_Data                |                                                                                                 |
| Set Ai_Run_Data              | Set Node                   | Prepares AI run data for pricing                   | Get AI Usage Data              | Get AI Pricing                 |                                                                                                 |
| Get AI Pricing               | HTTP Request               | Retrieves live AI model pricing from Jina API     | Set Ai_Run_Data               | Get Models Price and Add Summary |                                                                                                 |
| Get Models Price and Add Summary | Code Node              | Calculates total cost and prepares summary data   | Get AI Pricing                | Google Sheets                 |                                                                                                 |
| Google Sheets               | Google Sheets Node          | Logs token usage and cost data to Google Sheets   | Get Models Price and Add Summary | None                         |                                                                                                 |
| Sticky Note                 | Sticky Note                | (Visual notes, no functional role)                 | None                          | None                          |                                                                                                 |
| Sticky Note6                | Sticky Note                | (Visual notes, no functional role)                 | None                          | None                          |                                                                                                 |
| Sticky Note1                | Sticky Note                | (Visual notes, no functional role)                 | None                          | None                          |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Points:**

   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’` with default settings.
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow` with default settings.

2. **Fetch Execution Data:**

   - Add an **n8n** node named `n8n Get Execution Data`.
   - Connect both trigger nodes (`When clicking ‘Test workflow’` and `When Executed by Another Workflow`) to this node.
   - Configure this node to accept an execution ID input (e.g., from trigger data or passed parameters).

3. **Extract AI Usage Data:**

   - Add a **Code** node named `Get AI Usage Data`.
   - Connect `n8n Get Execution Data` to this node.
   - Implement JavaScript code that parses the execution data JSON to extract:
     - Prompt tokens count
     - Completion tokens count
     - Models used
     - Tools used
   - Ensure robust error handling for missing or malformed data.

4. **Prepare AI Run Data:**

   - Add a **Set** node named `Set Ai_Run_Data`.
   - Connect `Get AI Usage Data` to this node.
   - Configure fields to store extracted token counts, models, and tools in variables for downstream use.

5. **Retrieve AI Pricing:**

   - Add an **HTTP Request** node named `Get AI Pricing`.
   - Connect `Set Ai_Run_Data` to this node.
   - Configure as follows:
     - HTTP Method: GET
     - URL: Jina AI pricing API endpoint (user must provide)
     - Authentication: Add Jina API Auth Header if available (optional)
     - On Error: Continue workflow execution (do not fail)
     - Retry on Fail: Enabled
   - This node fetches live pricing data for AI models.

6. **Calculate Cost and Prepare Summary:**

   - Add a **Code** node named `Get Models Price and Add Summary`.
   - Connect `Get AI Pricing` to this node.
   - Implement code that:
     - Maps models to their prices (using live data or fallback static prices)
     - Calculates cost for prompt and completion tokens
     - Aggregates total cost
     - Formats a JSON summary array with all relevant data

7. **Log Data to Google Sheets:**

   - Add a **Google Sheets** node named `Google Sheets`.
   - Connect `Get Models Price and Add Summary` to this node.
   - Configure:
     - Operation: Append Row
     - Spreadsheet ID and Sheet Name: Point to the user’s prepared Google Sheet with columns:
       `timestamp, Total Tokens, Prompt Tokens, Completion Tokens, Models Used, Tools Used, Total Cost, Json Array`
     - Map node fields to corresponding columns.
   - Set up Google OAuth2 credentials with appropriate permissions.

8. **Final Connections:**

   - Ensure the flow is:  
     Entry Trigger → n8n Get Execution Data → Get AI Usage Data → Set Ai_Run_Data → Get AI Pricing → Get Models Price and Add Summary → Google Sheets

9. **Testing:**

   - Use the manual trigger to test the workflow.
   - Verify that token usage and cost data are correctly appended to the Google Sheet.
   - Optionally, call this workflow as a sub-workflow from other workflows by passing the execution ID.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Image generated with ideoGener8r, a tool for AI-generated images.                                    | https://ideogener8r.com?utm_source=n8n&utm_medium=template&utm_campaign=tokenestim8r                   |
| Setup instructions include creating a Google Sheet with specific columns for logging token data.    | See section "🛠️ Setup Instructions" in workflow description                                          |
| Optional live pricing requires Jina AI API key and endpoint configuration in the HTTP Request node. | Jina AI API documentation (user must refer to official docs for API details)                          |
| This workflow is compatible with all n8n deployment types and uses only built-in nodes.             | Ensures broad usability without external dependencies                                                 |
| Suggestions for customization include changing logging destination, adding alerts, and aggregations.| See "🔧 How to customize this workflow to your needs" section                                         |

---

This document provides a complete, detailed reference to understand, reproduce, and modify the "Token Estim8r" workflow for AI token usage and cost tracking within n8n.