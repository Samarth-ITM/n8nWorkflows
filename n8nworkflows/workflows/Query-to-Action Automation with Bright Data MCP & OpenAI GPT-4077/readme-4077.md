Query-to-Action Automation with Bright Data MCP & OpenAI GPT

https://n8nworkflows.xyz/workflows/query-to-action-automation-with-bright-data-mcp---openai-gpt-4077


# Query-to-Action Automation with Bright Data MCP & OpenAI GPT

### 1. Workflow Overview

This workflow is a sophisticated AI-driven automation system designed to integrate Bright Data MCP tools with an OpenAI-powered chatbot interface. Its primary purpose is to enable users to interact naturally through chat messages to trigger complex backend data operations without needing API knowledge.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages from users.
- **1.2 AI Agent & Language Model Processing:** Uses OpenAI models to classify user intent, maintain conversational memory, and generate commands.
- **1.3 Tool Discovery & Classification:** Retrieves the list of available Bright Data MCP tools and matches user queries to the most suitable tool.
- **1.4 User Query Validation & Interaction:** Checks if additional information is needed from the user or if no tool matches the query, handling these cases gracefully.
- **1.5 Tool Execution:** Executes the selected MCP tool with the supplied parameters.
- **1.6 Memory & Output Handling:** Manages conversational context memory and prepares the final output to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block listens for incoming chat messages to trigger the workflow.
- **Nodes Involved:**  
  - When chat message received
  - AI Agent
- **Node Details:**

  - **When chat message received**
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`
    - Role: Entry point webhook for receiving chat messages.
    - Configuration: Uses a webhook ID to listen for incoming chat messages.
    - Inputs: External chat message via webhook.
    - Outputs: Passes message data to AI Agent.
    - Edge cases: Webhook misconfiguration or message format errors.
  
  - **AI Agent**
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Role: Central AI orchestration node managing language model, memory, and tools.
    - Configuration: Defaults; receives chat input and connects to language model and tools.
    - Inputs: Chat message from webhook.
    - Outputs: Commands and memory context for further processing.
    - Edge cases: AI service outages or input parsing issues.

---

#### 2.2 AI Agent & Language Model Processing

- **Overview:** Processes the user input with OpenAI models, manages conversation memory, and prepares the query for tool matching.
- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Simple Memory  
  - Execute the tool (as AI tool node)  
- **Node Details:**

  - **OpenAI Chat Model**
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - Role: Processes chat messages using OpenAI's GPT-4.1-nano.
    - Configuration: Model set to GPT-4.1-nano; receives user chat from AI Agent.
    - Inputs: Chat message.
    - Outputs: Processed chat output passed back to AI Agent.
    - Credentials: Requires OpenAI API key.
    - Edge cases: API rate limits, invalid keys, or model unavailability.

  - **Simple Memory**
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
    - Role: Maintains a sliding window of recent conversation history.
    - Configuration: Default buffer window.
    - Inputs: Chat messages.
    - Outputs: Contextual conversation memory to AI Agent.
    - Edge cases: Memory overflow, data loss on reset.

  - **Execute the tool**
    - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`
    - Role: Invokes a sub-workflow to execute the selected MCP tool.
    - Configuration: Points to a sub-workflow "Bright Data MCP Test" with parameters `query` and `session_id`.
    - Inputs: Query and session data from AI Agent.
    - Outputs: Tool execution results back to AI Agent.
    - Edge cases: Sub-workflow errors, parameter mismatches, tool execution failures.

---

#### 2.3 Tool Discovery & Classification

- **Overview:** Retrieves all available tools from Bright Data MCP and uses OpenAI to classify the user query to the appropriate tool.
- **Nodes Involved:**  
  - Tool call by the chatbot  
  - Bright Data MCP - List tools  
  - OpenAI  
  - If  
- **Node Details:**

  - **Tool call by the chatbot**
    - Type: `n8n-nodes-base.executeWorkflowTrigger`
    - Role: Triggers sub-workflow to fetch all available MCP tools.
    - Configuration: Passes input parameters `query` and `session_id`.
    - Inputs: Chat query.
    - Outputs: List of tools to next node.
    - Edge cases: Sub-workflow failure or timeouts.

  - **Bright Data MCP - List tools**
    - Type: `n8n-nodes-mcp.mcpClient`
    - Role: Calls Bright Data MCP API to list all available tools.
    - Configuration: Uses MCP client credentials.
    - Inputs: Triggered by tool call node.
    - Outputs: JSON list of available tools.
    - Credentials: MCP API key required.
    - Edge cases: Authentication failure, API rate limits.

  - **OpenAI**
    - Type: `@n8n/n8n-nodes-langchain.openAi`
    - Role: Classifies the user query by matching it to an MCP tool using AI.
    - Configuration: Uses GPT-4.1-nano model with a system prompt containing all tool schemas and instructions on expected output format (JSON with tool name, parameters, and additional info flag).
    - Inputs: User query and list of tools.
    - Outputs: JSON object with tool match results.
    - Credentials: OpenAI API key required.
    - Edge cases: Incorrect classification, missing parameters, API errors.

  - **If**
    - Type: `n8n-nodes-base.if`
    - Role: Checks if the AI classification returned a valid tool or "none".
    - Configuration: Condition to check if tool name is not "none".
    - Inputs: Classification result.
    - Outputs: Routes to either tool execution or error handling.
    - Edge cases: Logic errors or unexpected AI output format.

---

#### 2.4 User Query Validation & Interaction

- **Overview:** Determines if more input is needed from the user or if no matching tool is found, and handles these cases.
- **Nodes Involved:**  
  - If1  
  - Edit Fields1  
  - Return error message for no matching tool  
- **Node Details:**

  - **If1**
    - Type: `n8n-nodes-base.if`
    - Role: Checks if the AI output's `additional_info_needed` field is not "none".
    - Configuration: Condition on the `additional_info_needed` JSON property.
    - Inputs: Classification result from OpenAI.
    - Outputs: Routes to tool execution if no more info needed, or to prompt for more info.
    - Edge cases: Misinterpretation of AI flags, missing fields.

  - **Edit Fields1**
    - Type: `n8n-nodes-base.set`
    - Role: Stores the flag and information about needed user input in a field for further interaction.
    - Configuration: Assigns `needed_more_info_from_the_user` field with AI message content.
    - Inputs: AI classification output.
    - Outputs: Data to trigger user prompt.
    - Edge cases: Data overwriting or formatting issues.

  - **Return error message for no matching tool**
    - Type: `n8n-nodes-base.set`
    - Role: Creates a friendly error message when no tool matches the query.
    - Configuration: Sets a simple "No matching tool" message.
    - Inputs: From If node when no tool is found.
    - Outputs: Error message to user.
    - Edge cases: User confusion if repeated or unclear.

---

#### 2.5 Tool Execution

- **Overview:** Executes the selected MCP tool with parameters provided by the AI classification.
- **Nodes Involved:**  
  - Bright Data MCP - Execute a tool  
- **Node Details:**

  - **Bright Data MCP - Execute a tool**
    - Type: `n8n-nodes-mcp.mcpClient`
    - Role: Calls the Bright Data MCP API to execute the matched tool with user parameters.
    - Configuration: Dynamic tool name and parameters from AI classification output.
    - Inputs: Tool name and parameters JSON.
    - Outputs: Tool execution results JSON.
    - Credentials: MCP API key required.
    - Edge cases: API errors, invalid parameters, execution timeouts.

---

#### 2.6 Memory & Output Handling

- **Overview:** Manages conversation memory with AI and prepares the final response to the user.
- **Nodes Involved:**  
  - Chat Memory Manager  
  - Simple Memory1  
  - Copy the output from the MCP tool  
- **Node Details:**

  - **Chat Memory Manager**
    - Type: `@n8n/n8n-nodes-langchain.memoryManager`
    - Role: Inserts executed tool results into chat memory to provide context.
    - Configuration: Inserts JSON stringified tool results, hidden from UI.
    - Inputs: Tool execution results.
    - Outputs: Updated memory data.
    - Edge cases: Memory overflow, data corruption.

  - **Simple Memory1**
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
    - Role: Holds windowed chat memory using a session key for context continuity.
    - Configuration: Session key derived from `session_id`.
    - Inputs: Session ID and chat messages.
    - Outputs: Provides context for AI Agent.
    - Edge cases: Session ID mismatches, memory retention limits.

  - **Copy the output from the MCP tool**
    - Type: `n8n-nodes-base.set`
    - Role: Sets the raw output from the MCP tool as the final JSON output for use by the AI Agent.
    - Configuration: Outputs the `result` field from the tool execution node.
    - Inputs: Tool execution results.
    - Outputs: Final data sent back to AI Agent.
    - Edge cases: Missing or malformed tool results.

---

### 3. Summary Table

| Node Name                       | Node Type                              | Functional Role                          | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                  |
|--------------------------------|--------------------------------------|----------------------------------------|------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| When chat message received      | @n8n/n8n-nodes-langchain.chatTrigger| Entry webhook to receive chat messages | -                            | AI Agent                        |                                                                                              |
| AI Agent                       | @n8n/n8n-nodes-langchain.agent       | Main AI orchestration                   | When chat message received    | OpenAI Chat Model, Execute the tool  | ## AI Chat Agent You may use any AI model. Make sure to point the 'Execute the Tool' tool to the correct sub-workflow. |
| OpenAI Chat Model               | @n8n/n8n-nodes-langchain.lmChatOpenAi| Processes chat input with OpenAI model | AI Agent                     | AI Agent                        |                                                                                              |
| Simple Memory                  | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains recent conversation context  | OpenAI Chat Model            | AI Agent                        |                                                                                              |
| Execute the tool               | @n8n/n8n-nodes-langchain.toolWorkflow| Invokes sub-workflow to execute MCP tool | AI Agent                    | AI Agent                        |                                                                                              |
| Tool call by the chatbot        | n8n-nodes-base.executeWorkflowTrigger | Triggers sub-workflow to get MCP tools | -                            | Bright Data MCP - List tools    | ## Retrieve all possible tools To minimize the time to get all the tools, store all the results in Edit Field node or Code node. |
| Bright Data MCP - List tools    | n8n-nodes-mcp.mcpClient               | Retrieves all available MCP tools       | Tool call by the chatbot     | OpenAI                         |                                                                                              |
| OpenAI                        | @n8n/n8n-nodes-langchain.openAi       | Classifies user query to MCP tool       | Bright Data MCP - List tools | If                             | ## Classify the query and match it with the appropriate MCP tool. The tool will return a JSON object containing the tool’s name and its parameters, adhering to the schema of a specific tool. |
| If                            | n8n-nodes-base.if                     | Checks if a valid tool matched           | OpenAI                      | If1, Return error message for no matching tool | ## Verify the output Check the output from OpenAI if it needs some info from the user or there's no matching tool from their inquiry. |
| If1                           | n8n-nodes-base.if                     | Checks if more info is needed            | If                          | Bright Data MCP - Execute a tool, Edit Fields1 |                                                                                              |
| Edit Fields1                  | n8n-nodes-base.set                    | Stores flag indicating more info needed | If1                         | -                             |                                                                                              |
| Return error message for no matching tool | n8n-nodes-base.set                    | Returns "No matching tool" message        | If                          | -                             |                                                                                              |
| Bright Data MCP - Execute a tool| n8n-nodes-mcp.mcpClient               | Executes the matched MCP tool             | If1                         | Chat Memory Manager            |                                                                                              |
| Chat Memory Manager           | @n8n/n8n-nodes-langchain.memoryManager| Inserts tool results into chat memory    | Bright Data MCP - Execute a tool | Copy the output from the MCP tool | ## Deliver the output to the AI Agent                                                           |
| Simple Memory1                | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains session-based conversation memory | Chat Memory Manager        | Chat Memory Manager            |                                                                                              |
| Copy the output from the MCP tool | n8n-nodes-base.set                    | Sets final tool output as JSON for AI Agent | Chat Memory Manager         | AI Agent                      |                                                                                              |
| Sticky Note                   | n8n-nodes-base.stickyNote             | Notes                                  | -                            | -                             | ## AI Chat Agent You may use any AI model. Make sure to point the 'Execute the Tool' tool to the correct sub-workflow. |
| Sticky Note1                  | n8n-nodes-base.stickyNote             | Notes                                  | -                            | -                             | ## Retrieve all possible tools To minimize the time to get all the tools, store all the results in Edit Field node or Code node. |
| Sticky Note2                  | n8n-nodes-base.stickyNote             | Notes                                  | -                            | -                             | ## Classify the query and match it with the appropriate MCP tool. The tool will return a JSON object containing the tool’s name and its parameters, adhering to the schema of a specific tool. |
| Sticky Note3                  | n8n-nodes-base.stickyNote             | Notes                                  | -                            | -                             | ## Verify the output Check the output from OpenAI if it needs some info from the user or there's no matching tool from their inquiry. |
| Sticky Note4                  | n8n-nodes-base.stickyNote             | Notes                                  | -                            | -                             | ## Deliver the output to the AI Agent                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create webhook node: "When chat message received"**
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Configure webhook ID for incoming chat messages.
   - Position: Entry point of the workflow.

2. **Add AI Agent node: "AI Agent"**
   - Type: `@n8n/n8n-nodes-langchain.agent`
   - Connect input from "When chat message received".
   - Leave default options.
   - Connect outputs to "OpenAI Chat Model" and "Execute the tool".

3. **Add OpenAI Chat Model node: "OpenAI Chat Model"**
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
   - Model: GPT-4.1-nano (or substitute your OpenAI model).
   - Credentials: Link to valid OpenAI API key.
   - Connect input from "AI Agent".
   - Output back to "AI Agent".

4. **Add Simple Memory node: "Simple Memory"**
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
   - Default buffer window for conversation context.
   - Connect input from "OpenAI Chat Model".
   - Output to "AI Agent".

5. **Add Execute the tool node: "Execute the tool"**
   - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`
   - Configure with sub-workflow ID for "Bright Data MCP Test".
   - Map inputs: `query` from AI Agent's AI override input, `session_id` from JSON sessionId.
   - Connect input from "AI Agent".
   - Output back to "AI Agent".

6. **Create sub-workflow: "Bright Data MCP Test"**
   - Accept parameters: `query` (string), `session_id` (string).
   - This sub-workflow handles tool listing, classification, execution, and memory.
   
7. **Inside "Bright Data MCP Test" sub-workflow:**

   a. **Add "Tool call by the chatbot" node**
      - Type: `n8n-nodes-base.executeWorkflowTrigger`
      - Inputs: `query`, `session_id` as workflow inputs.
      - Output to "Bright Data MCP - List tools".

   b. **Add "Bright Data MCP - List tools" node**
      - Type: `n8n-nodes-mcp.mcpClient`
      - Credentials: MCP API key.
      - Output to "OpenAI".

   c. **Add "OpenAI" node**
      - Type: `@n8n/n8n-nodes-langchain.openAi`
      - Model: GPT-4.1-nano.
      - System prompt: Include all MCP tools with their schema.
      - Input message: User query.
      - Output to "If".

   d. **Add "If" node**
      - Condition: Check if tool `name` is not "none".
      - True output: "If1".
      - False output: "Return error message for no matching tool".

   e. **Add "If1" node**
      - Condition: Check if `additional_info_needed` is not "none".
      - True output: "Edit Fields1" (more info needed).
      - False output: "Bright Data MCP - Execute a tool".

   f. **Add "Edit Fields1" node**
      - Type: `n8n-nodes-base.set`
      - Save `needed_more_info_from_the_user` with AI message content.
      - Output: Ends or loops back to user input.

   g. **Add "Return error message for no matching tool" node**
      - Type: `n8n-nodes-base.set`
      - Set message: "No matching tool".
      - Output: Ends or notifies user.

   h. **Add "Bright Data MCP - Execute a tool" node**
      - Type: `n8n-nodes-mcp.mcpClient`
      - Credentials: MCP API key.
      - Tool name from AI output `message.content.name`.
      - Parameters from AI output `message.content.parameters` (converted to lowercase JSON string).
      - Output to "Chat Memory Manager".

   i. **Add "Chat Memory Manager" node**
      - Type: `@n8n/n8n-nodes-langchain.memoryManager`
      - Mode: Insert.
      - Insert tool execution results JSON string, hidden from UI.
      - Output to "Copy the output from the MCP tool" and "Simple Memory1".

   j. **Add "Simple Memory1" node**
      - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
      - Session key: Use `session_id` from chatbot.
      - Output to "Chat Memory Manager" (for context continuity).

   k. **Add "Copy the output from the MCP tool" node**
      - Type: `n8n-nodes-base.set`
      - Output: Set JSON output to tool execution result.
      - Output to AI Agent in main workflow.

8. **Credential Setup**

- Configure OpenAI API credentials in all OpenAI-related nodes.
- Configure Bright Data MCP API credentials in MCP client nodes.
- Ensure environment variable or direct input of API keys is secure.

9. **Connect all nodes as per the order above**

10. **Test with example queries, e.g., "current news or status about Taylor Swift"**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                     | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This template integrates Bright Data MCP tools with OpenAI GPT models to create a no-code AI agent capable of executing real-world data scraping and enrichment tasks via natural language chat.                                                                | Provided workflow description                                                                        |
| Installation of the MCP Community Node is required in n8n to enable MCP API calls. See: **Settings → Community Nodes → Search `n8n-nodes-mcp`**                                                                                                               | Workflow pre-requisites                                                                              |
| API keys must be securely stored and referenced in credentials for OpenAI and Bright Data MCP client nodes.                                                                                                                                                    | Setup instructions                                                                                   |
| The AI classifier prompt in OpenAI node is customizable to adjust tool matching logic and output format for different APIs or toolsets.                                                                                                                       | Customization notes                                                                                   |
| Memory nodes ensure conversation context is preserved, enabling multi-turn interactions with follow-up queries and maintaining session continuity.                                                                                                            | Workflow functionality                                                                               |
| Useful for automating complex scraping, lead generation, customer support, and internal workflow triggers via chat interfaces such as Slack, WhatsApp, or web chatbots.                                                                                         | Use cases                                                                                           |
| Video, branding, or additional help links are not included but can be added as sticky notes or external documentation in n8n.                                                                                                                                 | General workflow management                                                                          |

---

_Disclaimer: The provided content is exclusively derived from an automated n8n workflow, fully compliant with applicable content policies and legal requirements._