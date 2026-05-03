Enhance AI Prompts with GPT-4o-mini and Telegram Delivery

https://n8nworkflows.xyz/workflows/enhance-ai-prompts-with-gpt-4o-mini-and-telegram-delivery-3496


# Enhance AI Prompts with GPT-4o-mini and Telegram Delivery

### 1. Workflow Overview

This workflow is designed to enhance and optimize user-provided prompts by leveraging AI-powered natural language processing. Its primary use case is to take imprecise or vague prompts, improve their clarity, specificity, and formatting, and then deliver the optimized prompts back to the user via Telegram or pass them along for further automated processing.

The workflow’s logic is organized into the following functional blocks:

- **1.1 Input Reception:** Receives the initial prompt from an external trigger, typically another workflow.
- **1.2 AI Processing:** Uses an AI Agent with a configured OpenAI GPT-4o-mini model to enhance and refine the prompt with a detailed system message guiding the optimization.
- **1.3 Output Chunking:** Processes the AI-generated response to split it into Telegram-compatible message chunks, respecting character limits and formatting.
- **1.4 Delivery:** Sends the refined prompt back to the user via the Telegram messaging platform.
- **1.5 Memory Management:** Maintains conversational context or state using a simple memory buffer to improve AI interactions (optional and embedded within AI processing).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when called from another workflow, passing the user's prompt forward without modification.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - Type: ExecuteWorkflowTrigger  
    - Role: Acts as the entry point, receiving JSON input with the user’s initial prompt (under the key `query`) and Telegram chat metadata (like `chat_id`).  
    - Configuration: Uses passthrough input source to forward received data directly.  
    - Input: Triggered externally, e.g., by an upstream workflow.  
    - Output: Sends the JSON payload to the AI Agent node.  
    - Edge Cases: Missing or malformed input JSON could cause downstream errors; ensure the `query` field exists and is a string.  

#### 2.2 AI Processing

- **Overview:**  
  This block refines the initial prompt using an AI Agent powered by an OpenAI GPT model. The system message provides detailed instructions to enhance clarity, specificity, and formatting of the prompt.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory (context buffer)

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Receives the user prompt and system instructions; outputs an optimized prompt.  
    - Configuration:  
      - Input expression: `={{ $json.query }}` – extracts the original prompt from input JSON.  
      - System Message: Directs the model to enhance the prompt with clarity, specificity, detailed context, examples of output format (markdown), and guidance for code generation.  
      - Prompt Type: 'define' – sets up a defined prompt structure for the agent.  
      - Output parser enabled to format the response properly.  
    - Connections:  
      - Linked to OpenAI Chat Model as the language model backend.  
      - Linked to Simple Memory for conversation context.  
      - Outputs to Split into chunks1 node.  
    - Edge Cases:  
      - Model unavailability or API errors.  
      - Input prompt missing or empty.  
      - Response parsing errors if output format deviates.  
  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the GPT-4o-mini model for prompt enhancement.  
    - Configuration: Uses model named `gpt-4o-mini`.  
    - Requires valid OpenAI API credentials.  
    - Input: Text from AI Agent node.  
    - Output: AI-generated enhanced prompt for the agent.  
    - Edge Cases:  
      - API quota exceeded or network issues.  
      - Model version deprecated or changed behavior.  
  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains a sliding window of conversation history to provide context to the AI Agent.  
    - Configuration: Default settings, no parameters set explicitly.  
    - Input: Feeds memory context to AI Agent.  
    - Edge Cases:  
      - Memory overflow if conversation grows too large (unlikely here).  

#### 2.3 Output Chunking

- **Overview:**  
  Processes the AI-generated prompt to split the text into multiple message chunks suitable for Telegram delivery, respecting its character limits and formatting in markdown.

- **Nodes Involved:**  
  - Split into chunks1

- **Node Details:**  
  - **Split into chunks1**  
    - Type: Code (JavaScript)  
    - Role: Takes the full AI response and splits it into chunks of up to 3072 characters (Telegram message limit) without splitting words, prepending a header "# Optimized prompt" to the first chunk, and wrapping each chunk in markdown code blocks.  
    - Key Expressions: Uses JavaScript string manipulation with regex to replace multiple newlines and careful slicing to avoid word breaks.  
    - Input: AI Agent output JSON containing the enhanced prompt text.  
    - Output: An array of JSON objects each with a `text` field containing a markdown-formatted chunk.  
    - Edge Cases:  
      - Empty or very short AI output (outputs a single chunk).  
      - Special characters causing markdown syntax issues (escaped by backticks).  
      - Unexpected data formats from previous node.  

#### 2.4 Delivery

- **Overview:**  
  Sends the optimized prompt chunks back to the user via Telegram, using the chat ID received from the initial trigger.

- **Nodes Involved:**  
  - Telegram3

- **Node Details:**  
  - **Telegram3**  
    - Type: Telegram Node  
    - Role: Sends messages to a Telegram chat identified by `chat_id`.  
    - Configuration:  
      - Text content is dynamically populated from the chunked output (`={{ $json.text }}`).  
      - Chat ID is extracted from the original trigger node's JSON (`={{ $('When Executed by Another Workflow').item.json.chat_id }}`).  
      - On error, continues workflow execution to avoid halting on Telegram failures.  
      - Uses configured Telegram API credentials.  
    - Input: Receives message chunks from Split into chunks1.  
    - Output: None (end of chain).  
    - Edge Cases:  
      - Invalid or missing chat ID causes message failure.  
      - Telegram API limits or downtime.  
      - Message content exceeding limits (mitigated by chunking).  

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                     | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                                                            |
|----------------------------|----------------------------------|-----------------------------------|--------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | ExecuteWorkflowTrigger           | Workflow entry, input reception   | External trigger               | AI Agent                       | Trigger can be anything. For this example the trigger is a call from another workflow and a received Telegram message. Note integration possibility.  |
| AI Agent                   | LangChain Agent                   | Enhance user prompt with AI       | When Executed by Another Workflow, Simple Memory, OpenAI Chat Model | Split into chunks1             | Incoming trigger is processed by a LLM with a specific system prompt set aimed at improving the input prompt.                                          |
| OpenAI Chat Model          | LangChain OpenAI Chat Model       | GPT model for prompt enhancement  | AI Agent                      | AI Agent                      |                                                                                                                                                        |
| Simple Memory              | LangChain Memory Buffer Window    | Provide context memory to AI      | None (internal)                 | AI Agent                      |                                                                                                                                                        |
| Split into chunks1         | Code Node (JavaScript)            | Split AI output into Telegram-sized chunks | AI Agent                      | Telegram3                     | Improved prompt: Send as a response, Use as input for next nodes                                                                                       |
| Telegram3                  | Telegram Node                    | Deliver optimized prompt to user  | Split into chunks1             | None                         |                                                                                                                                                        |
| Sticky Note                | Sticky Note (Disabled)             | Documentation/comment node        | None                          | None                          | Trigger can be anything. For this example the trigger is a call from another workflow and a received Telegram message. Note integration possibility.  |
| Sticky Note1               | Sticky Note (Disabled)             | Documentation/comment node        | None                          | None                          | Incoming trigger is processed by a LLM with a specific system prompt set aimed at improving the input prompt.                                          |
| Sticky Note2               | Sticky Note (Disabled)             | Documentation/comment node        | None                          | None                          | Workflow documentation and setup instructions                                                                                                         |
| Sticky Note3               | Sticky Note (Disabled)             | Documentation/comment node        | None                          | None                          | Improved prompt: Send as a response, Use as input for next nodes                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add an **ExecuteWorkflowTrigger** node named `When Executed by Another Workflow`.  
   - Set input source to `passthrough` to accept raw JSON input (expects user prompt under `query` and `chat_id` for Telegram).  

2. **Create AI Model Node:**  
   - Add a **LangChain OpenAI Chat Model** node named `OpenAI Chat Model`.  
   - Select the model `gpt-4o-mini`.  
   - Configure OpenAI credentials with valid API key.  

3. **Create Memory Node:**  
   - Add a **LangChain Memory Buffer Window** node named `Simple Memory`.  
   - Use default settings (no parameters needed).  

4. **Create AI Agent Node:**  
   - Add a **LangChain Agent** node named `AI Agent`.  
   - Set input expression for text: `={{ $json.query }}` to extract user prompt.  
   - Configure the system message with detailed instructions to enhance the prompt, including clarity, specificity, formatting in markdown, and guidance for code generation.  
   - Set prompt type to `define`.  
   - Enable output parsing.  
   - Connect:  
     - `When Executed by Another Workflow` main output → `AI Agent` input.  
     - `OpenAI Chat Model` as the language model backend in AI Agent settings.  
     - `Simple Memory` as the AI Agent’s memory input.  

5. **Create Code Node to Split Output:**  
   - Add a **Code** node named `Split into chunks1`.  
   - Use the provided JavaScript code to split the AI output text into chunks of 3072 characters, adding a header and markdown formatting.  
   - Connect `AI Agent` output to `Split into chunks1` input.  

6. **Create Telegram Delivery Node:**  
   - Add a **Telegram** node named `Telegram3`.  
   - Set the message text to `={{ $json.text }}` to send each chunk.  
   - Set chat ID to `={{ $('When Executed by Another Workflow').item.json.chat_id }}` to send to the correct user.  
   - Configure Telegram API credentials with an authorized Telegram bot token.  
   - Configure error handling to continue on error to avoid workflow interruption.  
   - Connect `Split into chunks1` output to `Telegram3` input.  

7. **Activate the Workflow:**  
   - Ensure all credentials are properly configured and tested.  
   - Activate the workflow to start listening for triggers and processing inputs.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| This workflow can be integrated as a mid-step in larger workflows, triggered by other workflows or Telegram messages.                                                                                                         | Sticky Note on Trigger node                                        |
| The AI Agent uses a carefully crafted system prompt to ensure the optimized prompt is clear, detailed, and formatted in markdown, suitable for further automated use or human reading.                                        | Sticky Note on AI Agent node                                       |
| Telegram messages are split to respect limits and ensure readability, using markdown code blocks for formatting.                                                                                                              | Sticky Note on Split into chunks1 node                             |
| Ensure Telegram and OpenAI API credentials (API keys/tokens) are securely stored and valid to avoid authentication errors.                                                                                                   | Workflow setup instructions                                        |
| For customization, the AI model in the OpenAI Chat Model node can be changed to another supported GPT variant depending on your account and requirements.                                                                    | OpenAI Chat Model node settings                                    |
| The workflow design assumes input JSON includes `query` (the prompt) and `chat_id` (Telegram chat identifier) – ensure upstream workflows provide this structure.                                                             | Input Reception node expectations                                 |

---

This documentation fully describes the "Enhance AI Prompts with GPT-4o-mini and Telegram Delivery" workflow, its nodes, logic, and instructions for replication and maintenance.