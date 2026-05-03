Generate Written Content with GPT-4o Recursive Writing & Editing Agents

https://n8nworkflows.xyz/workflows/generate-written-content-with-gpt-4o-recursive-writing---editing-agents-3503


# Generate Written Content with GPT-4o Recursive Writing & Editing Agents

### 1. Workflow Overview

This workflow is designed for content creators and writers experimenting with recursive AI agent collaboration to generate and refine written content. It implements a multi-agent recursive framework where one AI agent writes a text blurb based on a topic, and a second AI agent edits the text. The process loops until the editor confirms the content is complete.

**Target Use Cases:**  
- Automated writing and editing with iterative improvements.  
- Recursive multi-agent AI workflows for content refinement.  
- Flexible integration with chat triggers or other input sources.

**Logical Blocks:**  
- **1.1 Input Reception:** Receives the initial user topic input from a chat message trigger.  
- **1.2 Edit Handling:** Retrieves and manages any previously suggested edits to support recursion.  
- **1.3 Writing Agent:** Generates or revises content based on the user input and edits.  
- **1.4 Editing Agent:** Reviews the writing agent’s output and suggests edits, including a completion status.  
- **1.5 Status Evaluation & Loop Control:** Checks if the editing is complete and either outputs the final result or recurses to the writing agent with new edits.  
- **1.6 Memory Management:** Maintains chat history context for agents to reference.  
- **1.7 Final Output:** Sends the approved content back to the chat interface.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures the user’s initial topic input from a chat message trigger node for processing.  
- **Nodes Involved:**  
  - `When chat message received`  
  - `chatInput`  

- **Node Details:**  
  - **When chat message received**  
    - Type: Langchain Chat Trigger  
    - Role: Entry point triggered by a chat message, capturing session ID and input text.  
    - Configuration: Default options; webhook ID set for external message reception.  
    - Input: External chat message (user’s topic).  
    - Output: JSON containing `chatInput` and session info.  
    - Potential Failures: Webhook unreachable, invalid message format, session ID missing.  
  - **chatInput**  
    - Type: Set node  
    - Role: Extracts and stores the chat input string into a variable named `chatInput`.  
    - Configuration: Sets `chatInput` = incoming JSON field `chatInput`.  
    - Input: Output from chat trigger.  
    - Output: JSON with a single field `chatInput`.  
    - Edge Cases: Missing or empty chatInput field, which would impact downstream nodes.

---

#### 2.2 Edit Handling

- **Overview:** Retrieves previous edit suggestions to support recursive editing or defaults to empty if none exist.  
- **Nodes Involved:**  
  - `handle edits`  

- **Node Details:**  
  - **handle edits**  
    - Type: Code node (JavaScript)  
    - Role: Checks if prior edit suggestions exist; sets `edits` to empty string if not.  
    - Configuration: Uses try-catch to safely access `edits` from the `set variables` node output.  
    - Key Expression: `edits = $node["set variables"].json["edits"] || ""`  
    - Input: Output from `chatInput` node initially, then from the `If Status Complete` node on recursion.  
    - Output: Object containing `edits` string.  
    - Edge Cases: Node `set variables` not executed yet, missing data, null or malformed edits string.  
    - Failure Modes: Expression errors, JavaScript exceptions handled gracefully by try-catch.

---

#### 2.3 Writing Agent

- **Overview:** Generates or revises a written blurb based on the user input topic and any suggested edits.  
- **Nodes Involved:**  
  - `Writing Agent`  
  - `OpenAI Chat Model` (as LLM backend)  
  - `Window Buffer Memory` (for chat context)  

- **Node Details:**  
  - **Writing Agent**  
    - Type: Langchain Agent node  
    - Role: Writer AI agent that creates or rewrites content incorporating edits.  
    - Configuration:  
      - System Message: "You are a writer. Write a short blurb about the topic given to you by the user."  
      - Prompt: Includes chat history and variables `chatInput` and `edits`.  
      - Output: The generated blurb text only, no additional commentary.  
    - Input: JSON containing `chatInput` and `edits` from `handle edits`.  
    - Output: JSON with `output` field containing the blurb.  
    - Connected to: Uses `OpenAI Chat Model` for generation; references `Window Buffer Memory` for context.  
    - Failure Modes: API rate limits, invalid prompts, empty input, timeout errors.  
  - **OpenAI Chat Model**  
    - Type: Langchain LLM node (OpenAI GPT-4o)  
    - Role: Provides language model capabilities for both Writing and Editing agents.  
    - Configuration: Model set to GPT-4o, credentials configured with OpenAI API.  
    - Input: Prompt from agents.  
    - Output: Text completion or chat response.  
    - Edge Cases: Credential expiration, API quota exhaustion, network failures.  
  - **Window Buffer Memory**  
    - Type: Langchain Memory Buffer (windowed)  
    - Role: Maintains limited chat history (last 10 turns) keyed by sessionId for context continuity.  
    - Configuration: Session key derived from `When chat message received` session ID, window length 10.  
    - Connected to: Feeds memory context into Writing and Editing agents.  
    - Edge Cases: Missing or invalid session ID, memory overflow.

---

#### 2.4 Editing Agent

- **Overview:** Reviews the writing agent’s output, suggests specific edits, and determines if the content is complete.  
- **Nodes Involved:**  
  - `Editing Agent`  
  - `Structured Output Parser`  
  - `set variables`  

- **Node Details:**  
  - **Editing Agent**  
    - Type: Langchain Agent node  
    - Role: Editor AI that critiques and suggests improvements as structured JSON.  
    - Configuration:  
      - System Prompt: "You are an editor..." with instructions to output JSON containing `status` ("complete"/"incomplete") and `edits` text.  
      - Input: Output text from Writing Agent (`output` field).  
      - Output: Structured JSON with properties `status` and `edits`.  
      - Output parser enabled to enforce JSON format.  
      - Max retries: 5 (to handle parsing or generation errors).  
    - Input: Writing Agent output.  
    - Output: JSON object with `status` and `edits`.  
    - Failure Modes: Incorrect JSON output, parsing errors, API failures.  
  - **Structured Output Parser**  
    - Type: Langchain Output Parser  
    - Role: Validates and parses editor's JSON output to ensure compliance with schema.  
    - Configuration: Manual schema specifying `status` (string) and `edits` (string).  
    - Input: Raw output from Editing Agent.  
    - Output: Parsed JSON object.  
    - Edge Cases: Invalid JSON, schema mismatches.  
  - **set variables**  
    - Type: Set node  
    - Role: Extracts `status` and `edits` from parsed JSON and stores them for flow control.  
    - Configuration: Assigns `status` and `edits` from Editing Agent output JSON.  
    - Input: Parsed JSON from Structured Output Parser.  
    - Output: JSON with `status` and `edits`.  
    - Edge Cases: Missing keys, empty values.

---

#### 2.5 Status Evaluation & Loop Control

- **Overview:** Determines if the editing process is complete and routes the workflow accordingly to either finish or recurse.  
- **Nodes Involved:**  
  - `If Status Complete`  
  - `chatOutput`  

- **Node Details:**  
  - **If Status Complete**  
    - Type: If node  
    - Role: Checks if `status` from Editing Agent equals "complete".  
    - Configuration: Condition: `$('Editing Agent').item.json.output.status == "complete"`  
    - Input: `set variables` output.  
    - Output:  
      - True branch: Proceed to output final content.  
      - False branch: Loop back to `handle edits` to pass new edits and recurse Writing Agent.  
    - Edge Cases: Missing status, unexpected values, case sensitivity issues.  
  - **chatOutput**  
    - Type: Set node  
    - Role: Prepares final approved blurb for output to chat.  
    - Configuration: Assigns `output` = Writing Agent’s final blurb text.  
    - Input: True branch from `If Status Complete`.  
    - Output: JSON with final content for chat response.  
    - Edge Cases: Empty or null text (should not happen if status is complete).

---

#### 2.6 Memory Management

- **Overview:** Maintains conversational context for the AI agents to produce coherent outputs across recursion cycles.  
- **Nodes Involved:**  
  - `Window Buffer Memory` (connected to Writing and Editing Agents)  

- **Node Details:**  
  - See 2.3 Writing Agent section for memory details.  
  - Ensures agents can "remember" past interactions within the session, improving relevance and consistency.

---

#### 2.7 Final Output

- **Overview:** Sends the finalized and approved text back to the chat interface for user consumption.  
- **Nodes Involved:**  
  - `chatOutput`  

- **Node Details:**  
  - See 2.5 Status Evaluation & Loop Control section.  
  - This node’s output connects directly to the chat interface’s response mechanism (implicit from trigger node).

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                                  | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                          |
|---------------------------|--------------------------------------|-------------------------------------------------|------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received| Langchain Chat Trigger                | Entry point receiving user topic input          | (external webhook)            | chatInput                      |                                                                                                    |
| chatInput                 | Set                                  | Extracts user input text into `chatInput`       | When chat message received    | handle edits                   |                                                                                                    |
| handle edits              | Code                                 | Retrieves previous edits or sets empty string   | chatInput, If Status Complete | Writing Agent                  |                                                                                                    |
| Writing Agent             | Langchain Agent                      | Generates/rewrites blurb based on topic & edits | handle edits                 | Editing Agent                  | ## Writing Agent\nThis agent writes and rewrites based on feedback from the editing agent.          |
| OpenAI Chat Model         | Langchain LLM (OpenAI GPT-4o)        | Provides language model for Writing & Editing   | Writing Agent, Editing Agent  | Writing Agent, Editing Agent   |                                                                                                    |
| Window Buffer Memory      | Langchain Memory Buffer Windowed      | Maintains chat history context for agents       | When chat message received    | Editing Agent, Writing Agent   |                                                                                                    |
| Editing Agent             | Langchain Agent                      | Reviews writing, suggests edits, outputs status | Writing Agent                | Structured Output Parser       | ## Editing Agent\nThis agent suggest edits to improve the writing output of the writing agent. It then evaluates whether the edits were incorporated into the writing. |
| Structured Output Parser  | Langchain Output Parser               | Parses editor JSON output                         | Editing Agent                | set variables                 |                                                                                                    |
| set variables             | Set                                  | Stores `status` and `edits` from editor output  | Structured Output Parser      | If Status Complete             |                                                                                                    |
| If Status Complete        | If                                   | Checks if editing is complete, controls loop    | set variables                | chatOutput (true), handle edits (false) |                                                                                                    |
| chatOutput                | Set                                  | Outputs final approved blurb to chat             | If Status Complete (true)     | (chat interface response)      |                                                                                                    |
| Sticky Note               | Sticky Note                          | Informational note: Writing Agent description    |                              |                                 | ## Writing Agent\nThis agent writes and rewrites based on feedback from the editing agent.          |
| Sticky Note1              | Sticky Note                          | Informational note: Editing Agent description    |                              |                                 | ## Editing Agent\nThis agent suggest edits to improve the writing output of the writing agent. It then evaluates whether the edits were incorporated into the writing. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add `Langchain Chat Trigger` node named "When chat message received".  
   - Configure webhook to receive chat messages with a session ID and text input.

2. **Extract Chat Input:**  
   - Add `Set` node named "chatInput".  
   - Set variable `chatInput` = `{{$json.chatInput}}` from trigger output.

3. **Prepare Edit Handling:**  
   - Add `Code` node named "handle edits".  
   - JavaScript code:  
     ```javascript
     let edits = "";
     try {
       edits = $node["set variables"].json["edits"] || "";
     } catch (err) {
       edits = "";
     }
     return { edits };
     ```  
   - Input: Connect from "chatInput" for first run, and from "If Status Complete" false branch on recursion.

4. **Add Language Model Node:**  
   - Add `Langchain LLM` node named "OpenAI Chat Model".  
   - Select model: GPT-4o.  
   - Set credentials to your OpenAI API key.

5. **Configure Memory Node:**  
   - Add `Langchain Memory Buffer Window` node named "Window Buffer Memory".  
   - Session key: `={{ $('When chat message received').item.json.sessionId }}`  
   - Context window length: 10.  
   - Connect memory output to both Writing and Editing Agents.

6. **Add Writing Agent:**  
   - Add `Langchain Agent` node named "Writing Agent".  
   - Prompt:  
     ```
     Always review chat history.

     Write a blurb based on my input topic: {{ $('chatInput').item.json.chatInput }}

     If there are any suggested edits, make sure to incorporated them into the blurb: {{ $json.edits }}

     ONLY OUTPUT THE BLURB, NO ADDITIONAL WORDS.
     ```  
   - System Message: "You are a writer. Write a short blurb about the topic given to you by the user."  
   - Connect input from "handle edits".  
   - Set language model to "OpenAI Chat Model".  
   - Connect memory node for context.

7. **Add Editing Agent:**  
   - Add `Langchain Agent` node named "Editing Agent".  
   - Prompt:  
     ```
     You are an editor.

     Review the input and recommend specific edits to improve the writing.

     You are working with a writing agent that should implement your edits.

     Here are the variables that you output and what they mean:
     - status: either "complete" or "incomplete"
     - edits: your suggested edits.

     Use the structured output parser and output clean JSON in this format:
     {
       "status": "complete",
       "edits": "Description of edits or confirmation."
     }

     DO NOT INCLUDE ANY OTHER TEXT BESIDES THE JSON OUTPUT.

     Here is the input text from the writing agent:
     {{ $json.output }}
     ```  
   - Enable structured output parser.  
   - Max retries: 5.  
   - Connect input from "Writing Agent".  
   - Set language model to "OpenAI Chat Model".  
   - Connect memory node for context.

8. **Add Output Parser:**  
   - Add `Langchain Output Parser` node named "Structured Output Parser".  
   - Set schema manually: JSON with `status` (string) and `edits` (string).  
   - Connect input from "Editing Agent".  
   - Connect output to "set variables".

9. **Extract Variables:**  
   - Add `Set` node named "set variables".  
   - Assign `status` = `{{$json.status}}`  
   - Assign `edits` = `{{$json.edits}}`  
   - Connect input from "Structured Output Parser".  
   - Connect output to "If Status Complete".

10. **Add Conditional Node:**  
    - Add `If` node named "If Status Complete".  
    - Condition: Check if `$('Editing Agent').item.json.output.status` equals `"complete"`.  
    - True branch: Connect to "chatOutput".  
    - False branch: Connect back to "handle edits" to feed new edits and continue recursion.

11. **Prepare Final Output:**  
    - Add `Set` node named "chatOutput".  
    - Assign `output` = `{{$node["Writing Agent"].first().json.output}}`.  
    - Connect input from "If Status Complete" true branch.  
    - Final output should connect to the chat system’s response mechanism.

12. **Add Sticky Notes:**  
    - Add two Sticky Notes near Writing and Editing Agents describing their roles as per the summary table for documentation clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                      | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is a foundational template to experiment with recursive multi-agent AI writing and editing processes. Customize prompts to tailor style and depth.                                | Workflow description                                                                              |
| To use, import the workflow and configure OpenAI API credentials for GPT-4o or substitute another LLM compatible with Langchain agents.                                                        | Setup instructions                                                                              |
| Video walkthrough and template available at https://n8n.io/workflows/recursive-multi-agent-template                                                                                              | n8n official workflow template page                                                             |
| Sticky notes in the workflow clarify the roles of the Writing Agent and Editing Agent for easier maintenance and onboarding.                                                                     | Workflow node sticky notes                                                                       |
| Ensure API limits and token quotas fit your usage patterns to avoid unexpected failures during recursive loops.                                                                                   | Operational advice                                                                              |
| Recursive loops rely on reliable session ID propagation and memory context; verify webhook and session management for smooth operation.                                                          | Technical note on session and memory handling                                                   |
| Editing Agent outputs strictly JSON; any deviation or malformed output breaks the loop — consider adding error handling or validation enhancements if extending the workflow.                    | Potential extension point                                                                        |

---

**Disclaimer:**  
The text above is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.