Track OpenAI Token Usage and AI Agent Metrics with Google Sheets Dashboard

https://n8nworkflows.xyz/workflows/track-openai-token-usage-and-ai-agent-metrics-with-google-sheets-dashboard-7231


# Track OpenAI Token Usage and AI Agent Metrics with Google Sheets Dashboard

### 1. Workflow Overview

This workflow is designed to track OpenAI (or other LLM provider) token usage and AI agent metrics, logging them into Google Sheets for observability and cost monitoring. It is particularly useful for teams or developers running AI agents who want to monitor token consumption, costs, and tool usage with client and workflow metadata.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception:** Captures incoming chat messages via a LangChain chat trigger.
- **1.2 Metadata Enrichment:** Adds workflow, execution, and client identifiers to the data.
- **1.3 AI Processing & Token Tracking:** Runs the AI agent with a LangChain ChatOpenAI model that includes a callback to track token usage and compute costs.
- **1.4 Branching on Tool Usage:** Determines if a tool was used in the AI agent's intermediate steps.
- **1.5 Logging Metrics to Google Sheets:** Logs detailed token usage metrics and tool usage into two separate Google Sheets tabs for cost tracking and observability.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming chat messages from users to trigger the workflow.
- **Nodes Involved:** 
  - When chat message received

##### Node: When chat message received
- **Type:** `@n8n/n8n-nodes-langchain.chatTrigger` (LangChain Chat Trigger)
- **Technical Role:** Entry point for user chat messages, launching the workflow.
- **Configuration:** Public webhook enabled; initial system message welcoming the user and asking if they are an existing client.
- **Key Expressions/Variables:** None (static initial message).
- **Input:** External chat messages via webhook.
- **Output:** Passes chat input to next node.
- **Edge Cases:** 
  - Webhook connectivity issues.
  - Unauthorized or malformed chat payloads.
- **Version Requirements:** Type version 1.1.
- **Sub-workflows:** None.

#### 2.2 Metadata Enrichment

- **Overview:** Adds workflow ID, execution ID, and client ID to the data for traceability.
- **Nodes Involved:** 
  - Set • Workflow + Client Metadata

##### Node: Set • Workflow + Client Metadata
- **Type:** `n8n-nodes-base.set`
- **Technical Role:** Attaches contextual metadata to each execution.
- **Configuration:** 
  - `workflow_id` set dynamically from `$workflow.id`.
  - `execution_id` set dynamically from `$execution.id`.
  - `client_id` hardcoded as `"123"` (should be customized).
- **Key Expressions:** `={{ $workflow.id }}`, `={{ $execution.id }}`, `"123"`.
- **Input:** Incoming chat message data.
- **Output:** Passes enriched data to AI Agent node.
- **Edge Cases:**
  - Missing or incorrect workflow/execution context.
  - Hardcoded client_id may not be suitable for multi-client environments.
- **Version Requirements:** Type version 3.4.
- **Sub-workflows:** None.

#### 2.3 AI Processing & Token Tracking

- **Overview:** Processes the chat input with an AI agent using LangChain, tracking token usage and costs live via a custom callback, then passes the AI output forward.
- **Nodes Involved:**
  - LangChain Chat Model + Token Callback
  - AI Agent

##### Node: LangChain Chat Model + Token Callback
- **Type:** `@n8n/n8n-nodes-langchain.code`
- **Technical Role:** Defines a LangChain ChatOpenAI model instance with a custom callback to log token usage and costs.
- **Configuration:** 
  - Uses environment variable `OPENAI_API_KEY` for authentication.
  - Model name set to `"gpt-5-mini-2025-08-07"`.
  - Token costs per million tokens: input = 0.25 USD, output = 2.00 USD.
  - Callback function extracts token usage metadata from the LLM response.
  - Calls a Google Sheets tool node (`googleSheetTool.func(row)`) to append token usage data as a row.
  - Retrieves metadata (`workflow_id`, `execution_id`, `client_id`) from "Set metadata" node.
- **Key Expressions/Variables:** Environment variables, LangChain API usage metadata, callback function with async Google Sheets call.
- **Input:** Expects an AI tool connection (Google Sheets tool node).
- **Output:** Produces an AI language model output passed to the AI Agent node.
- **Edge Cases:**
  - Missing or invalid API key environment variable.
  - Network or API errors from OpenAI or Google Sheets.
  - Callback failure resulting in lost logging data.
  - Model name or cost misconfiguration causing inaccurate cost tracking.
- **Version Requirements:** Type version 1.
- **Sub-workflows:** None.

##### Node: AI Agent
- **Type:** `@n8n/n8n-nodes-langchain.agent`
- **Technical Role:** Runs the AI agent using the chat input text and the configured LangChain model from the previous node.
- **Configuration:** 
  - Receives chat input text from "When chat message received".
  - System message: "You are a helpful assistant".
  - Returns intermediate steps for decision branching.
  - Prompt type is "define".
- **Key Expressions:** `={{  $('When chat message received').item.json.chatInput }}`.
- **Input:** Chat input and AI model instance.
- **Output:** AI response JSON with intermediate steps.
- **Edge Cases:** 
  - AI model errors or timeouts.
  - Empty or malformed chat inputs.
  - Excessive intermediate steps causing performance issues.
- **Version Requirements:** Type version 2.1.
- **Sub-workflows:** None.

#### 2.4 Branching on Tool Usage

- **Overview:** Checks if the AI agent used an external tool during its processing by inspecting intermediate steps.
- **Nodes Involved:** 
  - Branch • Tool Used?

##### Node: Branch • Tool Used?
- **Type:** `n8n-nodes-base.if`
- **Technical Role:** Conditional routing based on whether intermediate steps array is empty.
- **Configuration:** 
  - Checks if `intermediateSteps` array is empty.
  - If empty: no tool used; no further action.
  - If not empty: tool used; proceed to logging.
- **Key Expressions:** `={{ $json.intermediateSteps }}` checked for emptiness.
- **Input:** AI Agent output node.
- **Output:** 
  - True branch: no output (ends).
  - False branch: continues to logging node.
- **Edge Cases:** 
  - Missing intermediateSteps property.
  - Unexpected data formats causing false negatives.
- **Version Requirements:** Type version 2.2.
- **Sub-workflows:** None.

#### 2.5 Logging Metrics to Google Sheets

- **Overview:** Two Google Sheets nodes log token usage and AI agent metrics for observability and cost tracking.

- **Nodes Involved:**
  - Token Usage Log
  - Log • Token Metrics to Sheets (Tool)

##### Node: Token Usage Log
- **Type:** `n8n-nodes-base.googleSheetsTool`
- **Technical Role:** Appends detailed token usage and cost data to a Google Sheet tab named "Token cost tracker".
- **Configuration:** 
  - Automatically maps input data fields for date, model, workflow and execution IDs, client ID, token counts, costs, etc.
  - Uses Google Sheets OAuth2 credentials.
  - Targets spreadsheet with ID left blank (user must configure).
  - Sheet tab identified by `gid=0` (default first sheet).
- **Key Expressions:** Uses `$fromAI(...)` to extract values passed from AI callback.
- **Input:** Data object constructed in the LangChain token callback.
- **Output:** None (append operation).
- **Edge Cases:** 
  - Missing or incorrect Google Sheets document or sheet.
  - OAuth2 token expiration or invalid credentials.
  - Network errors during append operation.
- **Version Requirements:** Type version 4.5.
- **Sub-workflows:** None.

##### Node: Log • Token Metrics to Sheets (Tool)
- **Type:** `n8n-nodes-base.googleSheets`
- **Technical Role:** Logs AI agent observability data including input, output, tool usage, and metadata.
- **Configuration:** 
  - Defines columns: date, input, output, tool used, client_id, workflow_id, execution_id.
  - Appends rows to Google Sheet tab named "Observability" (`gid=94625050`).
  - Uses Google Sheets OAuth2 credentials.
  - Spreadsheet ID is set to `1O4PVOD584KWLzzj8gEym95hs8z3-Y4UjImMwKfby98c`.
- **Key Expressions:**
  - Date: current timestamp `$now`.
  - Input from chat message node.
  - Output and tool usage from AI Agent output.
  - Metadata from "Set • Workflow + Client Metadata".
- **Input:** From the false branch of the branch node (tool used).
- **Output:** None (append operation).
- **Edge Cases:** 
  - Google Sheets API quota exceeded.
  - Data format mismatches.
  - Credential expiration.
- **Version Requirements:** Type version 4.6.
- **Sub-workflows:** None.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                        | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                           |
|----------------------------------|----------------------------------|-------------------------------------|-------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received        | LangChain Chat Trigger            | Receives incoming chat messages      | (Webhook)                     | Set • Workflow + Client Metadata | **Start here (required settings)** See sticky note for setup instructions, environment variables, and sheet configuration.            |
| Set • Workflow + Client Metadata | Set                              | Adds workflow, execution, client IDs | When chat message received     | AI Agent                       | See sticky note for client_id customization and metadata usage.                                                                        |
| LangChain Chat Model + Token Callback | LangChain Code (JS)             | Configures LLM with token usage callback | Token Usage Log (Google Sheets Tool) | AI Agent                       | See sticky note for model-specific tweaks and token cost configuration.                                                                |
| AI Agent                         | LangChain Agent                   | Runs AI agent with chat input        | Set • Workflow + Client Metadata, LangChain Chat Model + Token Callback | Branch • Tool Used?             |                                                                                                                                        |
| Branch • Tool Used?               | If                               | Checks if AI agent used a tool       | AI Agent                      | Log • Token Metrics to Sheets (Tool) (false branch) |                                                                                                                                        |
| Token Usage Log                  | Google Sheets Tool                | Logs token usage and cost data       | LangChain Chat Model + Token Callback |                                | See sticky note for spreadsheet and sheet selection.                                                                                   |
| Log • Token Metrics to Sheets (Tool) | Google Sheets                   | Logs AI agent input/output metrics   | Branch • Tool Used? (false branch) |                                | See sticky note for spreadsheet and sheet selection.                                                                                   |
| Sticky Note8                    | Sticky Note                      | Tutorial video and starting instructions | None                        | None                           | Link to YouTube video walkthrough for building AI personal assistant: https://youtu.be/JSulRS128MA                                     |
| Sticky Note                     | Sticky Note                      | Setup instructions and workflow summary | None                        | None                           | Detailed setup instructions, environment variables, credential setup, and customization tips.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create node:** `When chat message received` (LangChain Chat Trigger)
   - Set `public` to true.
   - Add initial welcome message:  
     ```
     Welcome to Troopers!

     So glad you’re here! 😊 Are you already a Troopers client, or just getting started?
     ```
   - Ensure webhook URL is accessible.

2. **Create node:** `Set • Workflow + Client Metadata` (Set)
   - Add three fields:
     - `workflow_id` = `={{ $workflow.id }}`
     - `execution_id` = `={{ $execution.id }}`
     - `client_id` = `"123"` (replace with your own client identifier)
   - Connect output from "When chat message received" to this node.

3. **Create node:** `Token Usage Log` (Google Sheets Tool)
   - Choose operation `append`.
   - Connect credentials: Google Sheets OAuth2.
   - Set Spreadsheet ID (your own).
   - Set Sheet Name or GID to `"gid=0"` or the appropriate tab for token cost tracking.
   - Enable `autoMapInputData`.
   - Define columns to map: date, model, llm_node, client_id, input_cost, total_cost, output_cost, workflow_id, execution_id, input_tokens, total_tokens, output_tokens, input_token_cost, output_token_cost.
   - Leave documentId blank for now to configure later.

4. **Create node:** `LangChain Chat Model + Token Callback` (LangChain Code)
   - Language: JavaScript.
   - Paste the provided code that:
     - Requires `ChatOpenAI` from `@langchain/openai`.
     - Reads `OPENAI_API_KEY` from env vars.
     - Sets model to `"gpt-5-mini-2025-08-07"`.
     - Sets input token cost = 0.25, output token cost = 2.0.
     - Defines callback to extract token usage and cost.
     - Calls `googleSheetTool.func(row)` using the Google Sheets Tool node created above.
   - Connect the `ai_tool` input of this node to the "Token Usage Log" node.
   - Connect output to the AI Agent node (to be created next).

5. **Create node:** `AI Agent` (LangChain Agent)
   - Set parameter `text` to `={{ $('When chat message received').item.json.chatInput }}`.
   - Under options, set `systemMessage` to `"You are a helpful assistant"`.
   - Enable `returnIntermediateSteps`.
   - Set `promptType` to `"define"`.
   - Connect input from "Set • Workflow + Client Metadata" (main output).
   - Connect `ai_languageModel` input from "LangChain Chat Model + Token Callback".

6. **Create node:** `Branch • Tool Used?` (If)
   - Condition: Check if `{{$json.intermediateSteps}}` is empty.
     - If empty: no tool used.
     - If not empty: tool used.
   - Connect input from "AI Agent".

7. **Create node:** `Log • Token Metrics to Sheets (Tool)` (Google Sheets)
   - Operation: Append.
   - Connect credentials: Google Sheets OAuth2.
   - Set Spreadsheet ID to `"1O4PVOD584KWLzzj8gEym95hs8z3-Y4UjImMwKfby98c"`.
   - Set Sheet Name or GID to `94625050` (Observability sheet).
   - Define columns: date (`{{$now}}`), input (from chat input), output (from AI agent output), Tool use (from intermediate step tool name), client_id, workflow_id, execution_id.
   - Connect input from the false branch output of "Branch • Tool Used?".

8. **Connect all nodes sequentially:**
   - When chat message received → Set • Workflow + Client Metadata → AI Agent
   - LangChain Chat Model + Token Callback (input from Token Usage Log) → AI Agent (ai_languageModel input)
   - AI Agent → Branch • Tool Used?
   - Branch false output → Log • Token Metrics to Sheets (Tool)
   - Token Usage Log → LangChain Chat Model + Token Callback (ai_tool input)

9. **Credentials:**
   - Configure Google Sheets OAuth2 credentials for both Google Sheets nodes.
   - Ensure `OPENAI_API_KEY` (or equivalent) is set as an environment variable in n8n or server environment.

10. **Customize:**
    - Replace `client_id` in Set node as needed.
    - Update Spreadsheet IDs and Sheet Names to your own Google Sheets documents.
    - Adjust model name and token costs in LangChain code node to reflect your provider’s pricing.
    - Optionally add rate limiting or input validation as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                   | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Start here: Step-by-step YouTube Tutorial for building an AI personal assistant with LangChain and n8n.                                                                                        | https://youtu.be/JSulRS128MA                                                                           |
| **Setup instructions:** Set `client_id`, `workflow_id`, and `execution_id` in metadata node; configure token costs and model in LangChain code node; add Google Sheets OAuth2 credentials; replace spreadsheet IDs and sheet names. | Sticky Note nodes in workflow                                                                          |
| Workflow Purpose: Receives chat messages, runs AI agent, tracks tokens and costs, logs metrics to Google Sheets with metadata for observability and cost control.                             | Workflow description and node configurations                                                           |
| Customize token prices and model parameters according to your AI provider’s pricing. Add more metadata or observability metrics (latency, request IDs) as needed.                            | Suggested extensions                                                                                   |
| Swap LangChain Chat Trigger for Webhook Trigger if chat UI is not required.                                                                                                                   | Sticky Note instructions                                                                                |

---

This completes the comprehensive reference for the "Track LLM Token Usage and Agent Observability on Google Sheets" workflow. It enables understanding, modification, and full reproduction without the raw JSON.