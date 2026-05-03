Reliable AI Agent Output  Without Structured Output Parser - w/ OpenAI & Switch

https://n8nworkflows.xyz/workflows/reliable-ai-agent-output--without-structured-output-parser---w--openai---switch-4316


# Reliable AI Agent Output  Without Structured Output Parser - w/ OpenAI & Switch

---

### 1. Workflow Overview

This workflow is designed to reliably obtain structured nutritional information from an AI agent based on a user-provided food item input, without relying on the built-in Structured Output Parser node, which is known to be unreliable with AI Agents. Instead, it uses OpenAI’s GPT-4.1-nano model and manual schema validation with retry logic to ensure the AI response strictly conforms to a predefined JSON schema.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception:** Receives chat messages containing food item queries via a webhook with basic authentication.
- **1.2 AI Processing:** Uses an AI Agent node powered by LangChain and OpenAI GPT-4.1-nano to generate nutritional data in JSON format based on the input.
- **1.3 Output Validation and Retry Control:** Parses and validates the AI output against the expected schema manually, sets a retry counter (`aiRunIndex`), and routes the flow based on validation results.
- **1.4 Output Routing:** Uses a Switch node to handle three scenarios — valid schema output, invalid schema with retry attempts, and fallback for unhandled cases.
- **1.5 Error Handling and Messaging:** Formats error prompts on schema validation failure, manages retry loops, and constructs user-facing messages in case retries are exhausted or output is invalid.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures incoming chat messages from external clients via a webhook with basic authentication. It serves as the entry point for the workflow.

**Nodes Involved:**  
- When chat message received

**Node Details:**

- **When chat message received**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger` (Webhook Trigger with chat mode)  
  - Configuration:  
    - Mode: webhook  
    - Public: true  
    - Response mode: responseNode (responds directly to webhook caller)  
    - Authentication: Basic Auth enabled  
  - Credentials: HTTP Basic Auth configured (credentials required externally)  
  - Input: HTTP chat message containing `chatInput` (food item query) and `sessionId`  
  - Output: Passes the chat input and session ID downstream for AI processing  
  - Edge cases: Invalid or missing authentication; malformed requests; missing chat input  
  - Version: 1.1

---

#### 2.2 AI Processing

**Overview:**  
This block uses a LangChain AI Agent node configured with OpenAI GPT-4.1-nano to generate nutritional data about the input food item. The prompt instructs the AI to return a JSON object strictly conforming to a specified schema or an error JSON if the input is invalid.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Simple Memory

**Node Details:**

- **AI Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Runs the AI agent with a system prompt to provide nutritional info in a strict JSON schema without structured output parser.  
  - Configuration:  
    - Text input: `={{ $json.chatInput }}` (from chat trigger)  
    - Max iterations: 10 (max internal AI steps)  
    - System message: Detailed instructions specifying exact JSON schema expected, including required fields (`alimentName`, `averageCalories`, `proteins`, `carbohydrates`, `sugar`, `fiber`, `fat`, `sodium`, `healthyScore`) and error response format.  
    - ReturnIntermediateSteps: true (to capture AI reasoning steps if needed)  
    - Prompt type: Define (custom prompt)  
  - AI Model: Connected to OpenAI Chat Model with GPT-4.1-nano model  
  - Input: Receives chatInput string  
  - Output: AI response JSON or error JSON string  
  - Edge cases: AI output might not conform to JSON schema; string output with code block wrappers; potential API timeouts or auth errors with OpenAI  
  - Version: 1.8

- **OpenAI Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: Provides the underlying GPT-4.1-nano model for the AI agent  
  - Configuration:  
    - Model: `gpt-4.1-nano`  
    - Temperature: 0.8 (balanced creativity)  
    - Response format: JSON object expected (but manually validated later)  
  - Credentials: OpenAI API key configured externally  
  - Input: Connected as language model for AI Agent  
  - Output: Provides AI-generated chat completions  
  - Edge cases: API quota limits, network issues, auth failures  
  - Version: 1.2

- **Simple Memory**  
  - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - Role: Maintains conversation history buffer for AI Agent to keep context  
  - Configuration: Default buffer window (no params specified)  
  - Input: Receives AI Agent output to maintain state  
  - Output: Feeds back to AI Agent as memory  
  - Edge cases: Memory overflow or loss of context if large conversations  
  - Version: 1.3

---

#### 2.3 Output Validation and Retry Control

**Overview:**  
This block manually parses and validates the AI Agent’s output against the required JSON schema, handling cases of invalid JSON, missing keys, type mismatches, and range errors. It also tracks the retry count (`aiRunIndex`) to limit repeated AI calls.

**Nodes Involved:**  
- Validate Output + Set `aiRunIndex`

**Node Details:**

- **Validate Output + Set `aiRunIndex`**  
  - Type: `n8n-nodes-base.set`  
  - Role: Parses AI output string (removing code block wrappers), validates JSON structure and types, sets retry index for loop control  
  - Configuration:  
    - Uses an Immediately Invoked Function Expression (IIFE) in an expression to:  
      - Parse JSON if output is a string with ```json wrapper  
      - Check for an allowed error object indicating invalid input food item  
      - Verify presence of all required keys  
      - Validate types for each key (string for name, numbers for nutrition fields)  
      - Validate range for `healthyScore` (0-10)  
      - Return either formatted JSON string or error object with specific error types (`invalid_json`, `missing_key`, `invalid_type`, `invalid_range`)  
    - Sets `aiRunIndex` to the current AI Agent run index (to track retries)  
  - Input: AI Agent’s raw output JSON or string  
  - Output: Parsed and validated output or error object, plus retry counter  
  - Edge cases: Parsing failures, missing keys, wrong types, out-of-range values, infinite retry risk if logic is changed  
  - On error: Continue regular output (does not stop workflow)  
  - Version: 3.4

---

#### 2.4 Output Routing

**Overview:**  
This block routes the workflow based on whether the AI output is valid, invalid but retryable, or an unhandled fallback. It ensures up to 4 retries and prevents infinite loops.

**Nodes Involved:**  
- Switch

**Node Details:**

- **Switch**  
  - Type: `n8n-nodes-base.switch`  
  - Role: Routes based on validation result and retry count  
  - Configuration:  
    - Three outputs with conditions:  
      - `invalidSchema`: triggers if `output.error` is defined and `aiRunIndex` < 3 (allows retries)  
      - `validSchema`: triggers if `output.alimentName` exists (valid output)  
      - `extra` (fallback): for any other case e.g. retries exhausted or unexpected data  
  - Input: Output from validation node  
  - Output connections:  
    - `invalidSchema` → Format Schema Error Prompt (retries AI Agent with error prompt)  
    - `validSchema` → Valid Schema Output (final success)  
    - `extra` → Set schemaValidationError & lastAgentOutput (handles failures)  
  - Edge cases: Incorrect modification can cause infinite loops or silent failures; retry count control critical  
  - Version: 3.2

---

#### 2.5 Error Handling and Messaging

**Overview:**  
Handles schema validation error messages by formatting a detailed error prompt for the AI Agent and providing fallback responses to the user if retries are exhausted. Also prepares the final validated nutritional data output.

**Nodes Involved:**  
- Format Schema Error Prompt  
- Valid Schema Output  
- Set schemaValidationError & lastAgentOutput  
- Set chat Output

**Node Details:**

- **Format Schema Error Prompt**  
  - Type: `n8n-nodes-base.set`  
  - Role: Creates a detailed schema error prompt string to send back to the AI Agent for retry  
  - Configuration:  
    - Assigns `schemaErrorPrompt` with a message that includes the expected JSON schema and instructions to fix the AI output  
    - Also assigns `sessionId` and `chatInput` for context  
  - Input: From `Switch` node’s invalidSchema output  
  - Output: To AI Agent for retry loop  
  - Edge cases: Misformatted error prompt can confuse AI agent  
  - Version: 3.4

- **Valid Schema Output**  
  - Type: `n8n-nodes-base.set`  
  - Role: Captures the valid nutritional data under `nutritionalValues` for downstream use or response  
  - Configuration:  
    - Sets `nutritionalValues` variable with validated output JSON object  
  - Input: From `Switch` node’s validSchema output  
  - Output: Ready for final response or further processing  
  - Version: 3.4

- **Set schemaValidationError & lastAgentOutput**  
  - Type: `n8n-nodes-base.set`  
  - Role: Stores last validation error and last AI output string for user messaging  
  - Configuration:  
    - Assigns variables `schemaValidationError` and `lastAgentOutput` from their respective previous nodes  
  - Input: From `Switch` node’s extra output (fallback)  
  - Output: To Set chat Output node for user-friendly message construction  
  - Version: 3.4

- **Set chat Output**  
  - Type: `n8n-nodes-base.set`  
  - Role: Constructs a user-facing output message combining schema validation error and last AI response as fallback context  
  - Configuration:  
    - Sets `output` string with explanatory text including `schemaValidationError` and `lastAgentOutput`  
  - Input: From previous set node  
  - Output: Final response to user indicating failure with context  
  - Version: 3.4

---

### 3. Summary Table

| Node Name                         | Node Type                                  | Functional Role                                      | Input Node(s)                  | Output Node(s)                              | Sticky Note                                                                                              |
|----------------------------------|--------------------------------------------|-----------------------------------------------------|-------------------------------|---------------------------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received        | @n8n/n8n-nodes-langchain.chatTrigger       | Receives incoming chat message via webhook          | -                             | AI Agent                                   |                                                                                                        |
| OpenAI Chat Model                 | @n8n/n8n-nodes-langchain.lmChatOpenAi      | Provides GPT-4.1-nano language model                 | -                             | AI Agent (language model)                    |                                                                                                        |
| Simple Memory                    | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation context for AI Agent          | AI Agent                      | AI Agent (memory)                            |                                                                                                        |
| AI Agent                        | @n8n/n8n-nodes-langchain.agent               | Generates nutritional info in JSON based on input    | When chat message received    | Validate Output + Set `aiRunIndex`           | See Sticky Note "AI Agent": Explains prompt and manual validation approach without structured parser    |
| Validate Output + Set `aiRunIndex` | n8n-nodes-base.set                          | Parses and validates AI output, sets retry counter   | AI Agent                      | Switch                                      | See Sticky Note "Validate Output + Set `aiRunIndex`": Explains manual schema validation and retries     |
| Switch                          | n8n-nodes-base.switch                        | Routes flow based on validation and retry count      | Validate Output + Set `aiRunIndex` | Format Schema Error Prompt / Valid Schema Output / Set schemaValidationError & lastAgentOutput | See Sticky Note "Switch": Warning about infinite loop risk and routing logic                             |
| Format Schema Error Prompt       | n8n-nodes-base.set                          | Creates error prompt for AI Agent retries             | Switch (invalidSchema)         | AI Agent                                   |                                                                                                        |
| Valid Schema Output              | n8n-nodes-base.set                          | Stores validated nutritional data cleanly             | Switch (validSchema)           | -                                           | See Sticky Note "Valid Schema Output": Explains storing clean validated output                          |
| Set schemaValidationError & lastAgentOutput | n8n-nodes-base.set                          | Stores last validation error and AI output for messaging | Switch (extra)                | Set chat Output                             | See Sticky Note "Output Handling (Valid & Invalid Schema)"                                             |
| Set chat Output                 | n8n-nodes-base.set                          | Constructs user-facing error message with fallback    | Set schemaValidationError & lastAgentOutput | -                                  | See Sticky Note "Output Handling (Valid & Invalid Schema)"                                             |
| Sticky Note                    | n8n-nodes-base.stickyNote                    | Documentation and warnings                            | -                             | -                                           | Multiple sticky notes provide detailed explanations for AI Agent, validation, switch logic, and output  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node:**
   - Name: `When chat message received`  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Set mode to `webhook`, enable `public`, set response mode to `responseNode`.  
   - Enable Basic Authentication and configure credentials (username/password).  
   - This node will receive JSON with `chatInput` and `sessionId`.

2. **Create an OpenAI Chat Model Node:**
   - Name: `OpenAI Chat Model`  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Select model: `gpt-4.1-nano`  
   - Set temperature to 0.8 for balanced responses.  
   - Set response format to `json_object`.  
   - Link OpenAI API credentials.

3. **Create a Simple Memory Node:**
   - Name: `Simple Memory`  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Use default configuration.  
   - Connect this node to feed conversation memory into the AI Agent node.

4. **Create an AI Agent Node:**
   - Name: `AI Agent`  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set `text` parameter to `={{ $json.chatInput }}` to use chat trigger input.  
   - Set `maxIterations` to 10.  
   - Copy and paste the detailed system prompt instructing the agent to return a JSON strictly matching the nutritional schema or a defined error JSON.  
   - Enable `returnIntermediateSteps`.  
   - Set prompt type to `define`.  
   - Connect the OpenAI Chat Model node as the language model and the Simple Memory node as memory.  
   - Connect the `When chat message received` node to this AI Agent node.

5. **Create a Set Node for Validation and Retry Tracking:**
   - Name: `Validate Output + Set aiRunIndex`  
   - Type: `n8n-nodes-base.set`  
   - Add an assignment for `output` using the provided IIFE JavaScript expression that:  
     - Parses AI output string (handles code block wrappers)  
     - Validates presence and types of all required keys  
     - Checks `healthyScore` range 0-10  
     - Returns formatted JSON or error object  
   - Add an assignment for `aiRunIndex` set to `={{ $node["AI Agent"].runIndex }}`  
   - Connect output of AI Agent node to this node.

6. **Create a Switch Node for Routing:**
   - Name: `Switch`  
   - Type: `n8n-nodes-base.switch`  
   - Add three outputs:  
     - `invalidSchema`: condition: `{{ $json.output.error !== undefined && $json.aiRunIndex < 3 }}` (boolean true)  
     - `validSchema`: condition: `{{ $json.output.alimentName }}` exists  
     - Fallback output: named `extra`  
   - Connect output of `Validate Output + Set aiRunIndex` node here.

7. **Create a Set Node for Formatting Schema Error Prompt:**
   - Name: `Format Schema Error Prompt`  
   - Type: `n8n-nodes-base.set`  
   - Assign `schemaErrorPrompt` a string containing the entire JSON schema and instructions for the AI to fix output (copy from prompt in original node).  
   - Assign `sessionId` and `chatInput` from the chat trigger node.  
   - Connect `Switch` node’s `invalidSchema` output here.

8. **Connect `Format Schema Error Prompt` output back to AI Agent node:**
   - This creates the retry loop for AI to correct its output.

9. **Create a Set Node for Valid Schema Output:**
   - Name: `Valid Schema Output`  
   - Type: `n8n-nodes-base.set`  
   - Assign `nutritionalValues` to `={{ $json.output }}` (clean validated data)  
   - Connect `Switch` node’s `validSchema` output here.

10. **Create a Set Node for Schema Validation Error and Last AI Output:**
    - Name: `Set schemaValidationError & lastAgentOutput`  
    - Type: `n8n-nodes-base.set`  
    - Assign `schemaValidationError` to `={{ $('Validate Output + Set `aiRunIndex`').item.json.output }}`  
    - Assign `lastAgentOutput` to `={{ $('AI Agent').item.json.output }}`  
    - Connect `Switch` node’s `extra` output here.

11. **Create a Set Node for Chat Output Message:**
    - Name: `Set chat Output`  
    - Type: `n8n-nodes-base.set`  
    - Assign `output` to a string combining `schemaValidationError` and `lastAgentOutput` to inform user of failure and provide last AI response.  
    - Connect from `Set schemaValidationError & lastAgentOutput` node.

12. **Ensure all connections match the original flow:**
    - `When chat message received` → `AI Agent`  
    - `AI Agent` → `Validate Output + Set aiRunIndex`  
    - `Validate Output + Set aiRunIndex` → `Switch`  
    - `Switch` invalidSchema → `Format Schema Error Prompt` → loops back to `AI Agent`  
    - `Switch` validSchema → `Valid Schema Output`  
    - `Switch` extra → `Set schemaValidationError & lastAgentOutput` → `Set chat Output`  

13. **Configure credentials:**
    - Set up OpenAI API key credentials for the OpenAI Chat Model node.  
    - Set up HTTP Basic Auth credentials for the webhook node.

14. **Set workflow execution timeout:**
    - Set global timeout to 60 seconds (or as needed).

15. **Test thoroughly:**
    - Test with valid food item inputs and invalid inputs to verify retry logic and error handling.  
    - Verify no infinite loops occur by checking `aiRunIndex` increments and limits retries to 4 attempts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow avoids the unreliable Structured Output Parser node by implementing manual JSON parsing and validation with retry logic. This approach has proven more reliable with OpenAI GPT-4.1 series models.                                                                                                                                                                                                                                                                                  | Core design principle explained in Sticky Note5.                                                                                                                                     |
| The Switch node is critical for controlling retry attempts and preventing infinite loops. Modifying its expressions without understanding can cause API credit waste or workflow crashes.                                                                                                                                                                                                                                                                                                        | See Sticky Note "Switch" for detailed warnings and explanation.                                                                                                                      |
| The AI Agent system prompt instructs the AI to provide nutritional information strictly in JSON according to a schema, or an error object if the input is invalid. The prompt also includes instructions for a schema error feedback loop.                                                                                                                                                                                                                                                    | See Sticky Note "AI Agent" for prompt details.                                                                                                                                         |
| Manual schema validation is implemented in a Set node using a JavaScript IIFE expression. This offers fine control over error types and retry logic without crashing the workflow on invalid AI outputs.                                                                                                                                                                                                                                                                                     | See Sticky Note "Validate Output + Set `aiRunIndex`".                                                                                                                                  |
| The workflow uses OpenAI GPT-4.1-nano model by default for best performance and schema adherence. Adjust this based on your API availability and cost considerations.                                                                                                                                                                                                                                                                                                                         | Model setting in OpenAI Chat Model node.                                                                                                                                               |
| For more information about the issues with Structured Output Parser node and recommended alternatives, see: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.outputparserstructured/common-issues/                                                                                                                                                                                                                                                                 | Official n8n documentation on Structured Output Parser issues.                                                                                                                       |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.