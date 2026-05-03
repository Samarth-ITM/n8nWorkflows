Scalable Multi-Agent Chat Using @mentions

https://n8nworkflows.xyz/workflows/scalable-multi-agent-chat-using--mentions-3473


# Scalable Multi-Agent Chat Using @mentions

---

### 1. Workflow Overview

This workflow enables scalable multi-agent chat conversations by engaging multiple uniquely configured AI agents within a single chat interface. It targets users who want to interact with several AI assistants simultaneously, each with distinct personalities, instructions, and underlying LLM models accessed via OpenRouter. The workflow supports triggering specific agents by `@mentions` or having all agents respond in randomized order if no mentions are present.

**Logical Blocks:**

- **1.1 Input Reception and Settings Initialization:** Receives user chat input and loads global and agent-specific configurations.
- **1.2 Mention Extraction and Agent Selection:** Parses the user message to detect `@mentions` and selects the corresponding agents or defaults to all agents in random order.
- **1.3 Agent Loop and Message Passing:** Iterates over selected agents, passing the user message or previous agent’s response sequentially to each agent.
- **1.4 AI Agent Invocation:** Dynamically configures and calls the AI agents with their specific system messages and models.
- **1.5 Conversation Memory Management:** Maintains conversation history for context continuity.
- **1.6 Response Aggregation and Formatting:** Collects all agent responses and combines them into a single formatted message for output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Settings Initialization

- **Overview:**  
  This block receives the chat message from the user and loads global user settings and agent configurations from centralized JSON code nodes.

- **Nodes Involved:**  
  - When chat message received  
  - Define Global Settings  
  - Define Agent Settings  

- **Node Details:**

  - **When chat message received**  
    - *Type:* Chat Trigger (LangChain)  
    - *Role:* Entry point for user chat input, triggers workflow on new message.  
    - *Configuration:* Default options; exposes `chatInput` and `sessionId`.  
    - *Input:* External chat message webhook.  
    - *Output:* User message JSON with `chatInput` and `sessionId`.  
    - *Edge Cases:* Missing or malformed input; webhook failures.

  - **Define Global Settings**  
    - *Type:* Code  
    - *Role:* Defines global user info and system message content common to all agents.  
    - *Configuration:* JSON object with user name, location, notes, and a global system message snippet.  
    - *Key Variables:*  
      - `user.name` (e.g., "Jon")  
      - `user.location` (e.g., "Melbourne, Australia")  
      - `user.notes` (e.g., "Jon likes a casual, informal conversation style.")  
      - `global.systemMessage` (e.g., "Don't overdo the helpful, agreeable approach.")  
    - *Input:* From chat trigger node.  
    - *Output:* JSON with global settings.  
    - *Edge Cases:* Invalid JSON or missing fields.

  - **Define Agent Settings**  
    - *Type:* Code  
    - *Role:* Defines all AI agents with their unique names, OpenRouter model identifiers, and system messages.  
    - *Configuration:* JSON object mapping agent names to their configurations, e.g.,  
      ```json
      {
        "Chad": { "name": "Chad", "model": "openai/gpt-4o", "systemMessage": "You are a helpful Assistant..." },
        "Claude": { "name": "Claude", "model": "anthropic/claude-3.7-sonnet", "systemMessage": "You are logical and practical." },
        "Gemma": { "name": "Gemma", "model": "google/gemini-2.0-flash-lite-001", "systemMessage": "You are super friendly and love to debate." }
      }
      ```  
    - *Input:* From Define Global Settings node.  
    - *Output:* JSON with agent configurations.  
    - *Edge Cases:* Empty agent list, malformed JSON.

---

#### 2.2 Mention Extraction and Agent Selection

- **Overview:**  
  Parses the user’s chat input to extract `@mentions` of agent names. If mentions exist, selects those agents in mention order; otherwise, selects all agents in randomized order.

- **Nodes Involved:**  
  - Extract mentions  
  - Loop Over Items  

- **Node Details:**

  - **Extract mentions**  
    - *Type:* Code  
    - *Role:* Parses the chat input string to find `@AgentName` mentions matching defined agents.  
    - *Configuration:*  
      - Reads `chatInput` from the chat trigger node.  
      - Reads agent names from `Define Agent Settings`.  
      - Uses regex to find mentions case-insensitively and in order of appearance.  
      - If no mentions found, returns all agents shuffled randomly.  
      - Outputs an array of agent objects with `name`, `model`, and `systemMessage`.  
    - *Key Expressions:* Uses regex `\B@(${agentNames})\b` for mention detection.  
    - *Input:* User message JSON, agent settings JSON.  
    - *Output:* Multiple items, each representing one selected agent.  
    - *Edge Cases:* No agents defined, no mentions found, mention names not matching exactly, empty input.  
    - *Failure Modes:* Regex errors, JSON parsing errors.

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Iterates over each selected agent item sequentially to process their response.  
    - *Configuration:* Default batch size (1), processes agents one by one.  
    - *Input:* Output array from Extract mentions node.  
    - *Output:* Iterative output to downstream nodes for each agent.  
    - *Edge Cases:* Empty input array (no agents), batch processing errors.

---

#### 2.3 Agent Loop and Message Passing

- **Overview:**  
  Controls the sequential flow of messages between agents. The first agent receives the original user message; subsequent agents receive the previous agent’s response as input, enabling a chain of replies.

- **Nodes Involved:**  
  - First loop? (If)  
  - Set user message as input  
  - Set last Assistant message as input  
  - Set lastAssistantMessage  

- **Node Details:**

  - **First loop?**  
    - *Type:* If  
    - *Role:* Checks if the current loop iteration is the first (index 0).  
    - *Configuration:* Condition: `$runIndex == 0` (true if first agent).  
    - *Input:* Loop Over Items output.  
    - *Output:*  
      - True path: first agent branch.  
      - False path: subsequent agents branch.  
    - *Edge Cases:* Loop index missing or invalid.

  - **Set user message as input**  
    - *Type:* Set  
    - *Role:* For the first agent, sets `chatInput` to the original user message.  
    - *Configuration:* Assigns `chatInput` = `When chat message received`.item.json.chatInput.  
    - *Input:* True branch of First loop? node.  
    - *Output:* JSON with `chatInput` for AI Agent.  
    - *Edge Cases:* Missing original chat input.

  - **Set last Assistant message as input**  
    - *Type:* Set  
    - *Role:* For subsequent agents, sets `chatInput` to the previous agent’s formatted response (`lastAssistantMessage`).  
    - *Configuration:* Assigns `chatInput` = `Set lastAssistantMessage`.first().json.lastAssistantMessage.  
    - *Input:* False branch of First loop? node.  
    - *Output:* JSON with `chatInput` for AI Agent.  
    - *Edge Cases:* Missing or empty previous assistant message.

  - **Set lastAssistantMessage**  
    - *Type:* Set  
    - *Role:* Formats the current agent’s response by prefixing with the agent’s name for clarity.  
    - *Configuration:* Assigns `lastAssistantMessage` = `**{{agentName}}**:\n\n{{agentResponse}}` using expressions referencing `Loop Over Items` and AI Agent output.  
    - *Input:* Output of AI Agent node.  
    - *Output:* JSON with formatted `lastAssistantMessage`.  
    - *Edge Cases:* Missing agent name or response text.

---

#### 2.4 AI Agent Invocation

- **Overview:**  
  Dynamically calls the AI agent with the current input message, system instructions, and model configuration. Uses OpenRouter to access various LLMs.

- **Nodes Involved:**  
  - AI Agent  
  - OpenRouter Chat Model  

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Sends the current input message to the selected AI agent with dynamic system message and model.  
    - *Configuration:*  
      - `text` input is set dynamically from `chatInput`.  
      - `systemMessage` is dynamically composed using:  
        - Current date/time  
        - User info from `Define Global Settings`  
        - List of all agent names  
        - Current agent’s name and system message from loop item  
        - Global system message  
      - Prompt type: define (custom prompt).  
    - *Input:* JSON with `chatInput` from Set nodes.  
    - *Output:* AI-generated response text.  
    - *Edge Cases:* API errors, invalid model names, authentication failures, timeout, rate limits.

  - **OpenRouter Chat Model**  
    - *Type:* LangChain OpenRouter LLM Chat Model  
    - *Role:* Provides the language model interface for AI Agent node.  
    - *Configuration:*  
      - `model` parameter dynamically set from current agent’s model name.  
      - Credentials: OpenRouter API key configured here.  
    - *Input:* From Extract mentions node (agent model).  
    - *Output:* Connected to AI Agent node as language model.  
    - *Edge Cases:* Credential misconfiguration, API quota exceeded.

---

#### 2.5 Conversation Memory Management

- **Overview:**  
  Maintains a sliding window memory of the conversation to provide context continuity across multiple turns.

- **Nodes Involved:**  
  - Simple Memory  

- **Node Details:**

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Stores conversation history keyed by session ID, providing context to AI Agent.  
    - *Configuration:*  
      - `sessionKey` set dynamically from chat trigger’s `sessionId`.  
      - `contextWindowLength`: 99 messages (configurable).  
    - *Input:* Conversation messages from AI Agent node.  
    - *Output:* Memory context injected into AI Agent calls.  
    - *Edge Cases:* Missing session ID, memory overflow, data persistence issues.

---

#### 2.6 Response Aggregation and Formatting

- **Overview:**  
  After all agents have responded, this block combines their formatted messages into a single output string separated by horizontal rules, ready to be sent back to the user.

- **Nodes Involved:**  
  - Combine and format responses  

- **Node Details:**

  - **Combine and format responses**  
    - *Type:* Code  
    - *Role:* Aggregates all `lastAssistantMessage` strings from loop output items into one formatted text block.  
    - *Configuration:*  
      - Extracts `lastAssistantMessage` from each item.  
      - Joins messages with `\n\n---\n\n` separator.  
      - Returns a single JSON item with combined `output` field.  
    - *Input:* All loop iteration outputs.  
    - *Output:* Single item with combined response text.  
    - *Edge Cases:* Missing or empty messages, large output size.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                          | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                      |
|----------------------------|--------------------------------------|----------------------------------------|------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received | Chat Trigger (LangChain)              | Entry point, receives user chat input  | (Webhook)                    | Define Global Settings         |                                                                                                |
| Define Global Settings      | Code                                 | Loads global user and system settings  | When chat message received   | Define Agent Settings          | ## Step 1: Configure Settings Nodes - Edit JSON to set user and system message info, agents.   |
| Define Agent Settings       | Code                                 | Loads agent configurations              | Define Global Settings       | Extract mentions              | ## Step 1: Configure Settings Nodes - Edit JSON to define agents, models, and system prompts.  |
| Extract mentions            | Code                                 | Parses chat input for @mentions         | Define Agent Settings        | Loop Over Items               |                                                                                                |
| Loop Over Items             | SplitInBatches                       | Iterates over selected agents           | Extract mentions             | Combine and format responses, First loop? |                                                                                                |
| First loop?                 | If                                   | Checks if current iteration is first    | Loop Over Items              | Set user message as input (true), Set last Assistant message as input (false) |                                                                                                |
| Set user message as input   | Set                                  | Sets chatInput to original user message | First loop? (true)           | AI Agent                     |                                                                                                |
| Set last Assistant message as input | Set                          | Sets chatInput to previous agent response | First loop? (false)          | AI Agent                     |                                                                                                |
| AI Agent                   | LangChain Agent                      | Sends message to AI with dynamic system message and model | Set user message as input, Set last Assistant message as input | Set lastAssistantMessage | ## Step 2: Connect Agent to OpenRouter - Set OpenRouter credentials, dynamic system messages.   |
| OpenRouter Chat Model       | LangChain OpenRouter LLM Chat Model | Provides LLM interface for AI Agent     | Extract mentions             | AI Agent (languageModel input) | ## Step 2: Connect Agent to OpenRouter - Set OpenRouter credentials.                            |
| Set lastAssistantMessage    | Set                                  | Formats agent response with agent name | AI Agent                    | Loop Over Items              |                                                                                                |
| Simple Memory              | LangChain Memory Buffer Window       | Maintains conversation history          | AI Agent                    | AI Agent (memory input)       |                                                                                                |
| Combine and format responses | Code                               | Aggregates all agent responses into one | Loop Over Items             | (Workflow output)             |                                                                                                |
| Sticky Note                | Sticky Note                         | Instructional notes                      |                              |                               | ## Step 1: Configure Settings Nodes - Edit JSON to set user and agent info.                    |
| Sticky Note1               | Sticky Note                         | Instructional notes                      |                              |                               | ## Step 2: Connect Agent to OpenRouter - Set OpenRouter credentials and dynamic parameters.    |
| Sticky Note2               | Sticky Note                         | Workflow title and branding              |                              |                               | # Scalable Multi-Agent Conversations                                                          |
| Sticky Note3               | Sticky Note                         | Workflow description and usage overview |                              |                               | ## About this workflow - Multi-agent chat with @mentions or all agents.                       |
| Sticky Note4               | Sticky Note                         | Workflow operation and limitations      |                              |                               | **How does it work?** - Settings, mentions, looping, memory, limitations.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node:**  
   - Add a `When chat message received` node (LangChain Chat Trigger).  
   - Configure webhook as needed.  
   - Output includes `chatInput` and `sessionId`.

2. **Create Global Settings Node:**  
   - Add a `Code` node named `Define Global Settings`.  
   - Paste JSON defining user info and global system message, e.g.:  
     ```js
     return [{
       json: {
         user: { name: "Jon", location: "Melbourne, Australia", notes: "Jon likes a casual, informal conversation style." },
         global: { systemMessage: "Don't overdo the helpful, agreeable approach." }
       }
     }];
     ```  
   - Connect `When chat message received` → `Define Global Settings`.

3. **Create Agent Settings Node:**  
   - Add a `Code` node named `Define Agent Settings`.  
   - Paste JSON defining agents, e.g.:  
     ```js
     return [{
       json: {
         Chad: { name: "Chad", model: "openai/gpt-4o", systemMessage: "You are a helpful Assistant. You are eccentric and creative..." },
         Claude: { name: "Claude", model: "anthropic/claude-3.7-sonnet", systemMessage: "You are logical and practical." },
         Gemma: { name: "Gemma", model: "google/gemini-2.0-flash-lite-001", systemMessage: "You are super friendly and love to debate." }
       }
     }];
     ```  
   - Connect `Define Global Settings` → `Define Agent Settings`.

4. **Create Extract Mentions Node:**  
   - Add a `Code` node named `Extract mentions`.  
   - Paste the provided JavaScript code that:  
     - Reads `chatInput` from `When chat message received`.  
     - Reads agent data from `Define Agent Settings`.  
     - Extracts `@mentions` or defaults to all agents shuffled.  
     - Outputs an array of agent objects with `name`, `model`, `systemMessage`.  
   - Connect `Define Agent Settings` → `Extract mentions`.

5. **Create Loop Over Items Node:**  
   - Add a `SplitInBatches` node named `Loop Over Items`.  
   - Default batch size 1 (process agents sequentially).  
   - Connect `Extract mentions` → `Loop Over Items`.

6. **Create First Loop? Node:**  
   - Add an `If` node named `First loop?`.  
   - Condition: `$runIndex == 0` (number equals 0).  
   - Connect `Loop Over Items` → `First loop?`.

7. **Create Set User Message as Input Node:**  
   - Add a `Set` node named `Set user message as input`.  
   - Assign `chatInput` = `When chat message received`.item.json.chatInput.  
   - Connect `First loop?` true output → `Set user message as input`.

8. **Create Set Last Assistant Message as Input Node:**  
   - Add a `Set` node named `Set last Assistant message as input`.  
   - Assign `chatInput` = `Set lastAssistantMessage`.first().json.lastAssistantMessage.  
   - Connect `First loop?` false output → `Set last Assistant message as input`.

9. **Create OpenRouter Chat Model Node:**  
   - Add a `LangChain OpenRouter Chat Model` node named `OpenRouter Chat Model`.  
   - Set `model` parameter to expression: `={{ $('Extract mentions').item.json.model }}`.  
   - Configure OpenRouter API credentials here.  
   - Connect `Extract mentions` → `OpenRouter Chat Model` (ai_languageModel input).

10. **Create AI Agent Node:**  
    - Add a `LangChain Agent` node named `AI Agent`.  
    - Set `text` parameter to `={{ $json.chatInput }}`.  
    - Set `systemMessage` parameter dynamically with expression combining:  
      - Current date/time  
      - User info from `Define Global Settings`  
      - List of all agent names  
      - Current agent name and system message from loop item  
      - Global system message  
    - Connect `Set user message as input` and `Set last Assistant message as input` → `AI Agent`.  
    - Connect `OpenRouter Chat Model` → `AI Agent` (language model input).  
    - Connect `Simple Memory` (see next step) → `AI Agent` (memory input).

11. **Create Simple Memory Node:**  
    - Add a `LangChain Memory Buffer Window` node named `Simple Memory`.  
    - Set `sessionKey` to `={{ $('When chat message received').first().json.sessionId }}`.  
    - Set `contextWindowLength` to 99.  
    - Connect `AI Agent` output → `Simple Memory` input (ai_memory).  
    - Connect `Simple Memory` output → `AI Agent` input (ai_memory).

12. **Create Set lastAssistantMessage Node:**  
    - Add a `Set` node named `Set lastAssistantMessage`.  
    - Assign `lastAssistantMessage` = `**{{ $('Loop Over Items').item.json.name }}**:\n\n{{ $json.output }}`.  
    - Connect `AI Agent` → `Set lastAssistantMessage`.

13. **Connect Set lastAssistantMessage back to Loop Over Items:**  
    - Connect `Set lastAssistantMessage` → `Loop Over Items` (to continue loop).

14. **Create Combine and Format Responses Node:**  
    - Add a `Code` node named `Combine and format responses`.  
    - Paste code to:  
      - Collect all `lastAssistantMessage` fields from loop output.  
      - Join them with `\n\n---\n\n` separator.  
      - Return a single JSON item with combined `output`.  
    - Connect `Loop Over Items` main output (after loop completes) → `Combine and format responses`.

15. **Final Output:**  
    - The combined output from `Combine and format responses` is the final aggregated response to send back to the user.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Step 1: Configure Settings Nodes - Edit JSON to set user and agent info, system messages, models | Sticky Note near Define Global Settings and Define Agent Settings nodes                          |
| Step 2: Connect Agent to OpenRouter - Set OpenRouter credentials and dynamic parameters          | Sticky Note near AI Agent and OpenRouter Chat Model nodes                                       |
| Scalable Multi-Agent Conversations                                                               | Workflow title sticky note                                                                      |
| About this workflow: Enables multi-agent chat with @mentions or all agents responding randomly   | Sticky Note near Extract mentions and Loop Over Items nodes                                     |
| How does it work? Settings, mentions, looping, memory, limitations                               | Sticky Note near First loop? and Loop Over Items nodes                                          |
| OpenRouter API credentials must be configured in the OpenRouter Chat Model node                  | Credential setup reminder                                                                       |
| Agents respond sequentially, not in parallel                                                    | Limitation noted in sticky notes and description                                               |
| User receives combined response only after all agents have replied                              | Limitation noted in sticky notes and description                                               |

---

This structured documentation provides a comprehensive understanding of the workflow’s design, node-by-node functionality, and stepwise reconstruction instructions, enabling advanced users and AI agents to reproduce, modify, or troubleshoot the multi-agent chat system effectively.