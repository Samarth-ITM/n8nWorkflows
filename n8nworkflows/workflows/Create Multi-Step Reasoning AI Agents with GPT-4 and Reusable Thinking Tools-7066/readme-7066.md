Create Multi-Step Reasoning AI Agents with GPT-4 and Reusable Thinking Tools

https://n8nworkflows.xyz/workflows/create-multi-step-reasoning-ai-agents-with-gpt-4-and-reusable-thinking-tools-7066


# Create Multi-Step Reasoning AI Agents with GPT-4 and Reusable Thinking Tools

### 1. Workflow Overview

This workflow implements an advanced AI agent reasoning framework using GPT-4 and reusable thinking tools within n8n. Its main purpose is to enable multi-step, structured reasoning by an AI agent through iterative internal "thought" processes before producing a final answer. This approach overcomes the limitation of a single-step think tool by repeatedly invoking a sub-workflow designed as a reusable "scratchpad" for planning and reflection.

**Target Use Cases:**  
- Complex conversational AI requiring stepwise reasoning and planning  
- Automations where AI must deliberate using multiple custom tools before responding  
- Scenarios demanding modular, reusable AI thinking workflows with adjustable prompts  

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving user chat messages to trigger AI processing.  
- **1.2 AI Agent Setup:** Configuring the AI agent node with system prompts and connecting memory, language model, and thinking tools.  
- **1.3 Thinking Tools:** Two reusable thinking tools ("Initial thoughts" and "Additional thoughts") implemented as calls to a sub-workflow for multi-step reasoning.  
- **1.4 AI Language Model & Memory:** OpenAI GPT-4 model node and a simple memory buffer to maintain context.  
- **1.5 Sub-Workflow Execution:** The reusable thinking sub-workflow invoked by the thinking tools.  
- **1.6 Documentation & Instructional Notes:** Sticky notes providing explanations, usage instructions, and customization guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives incoming chat messages which initiate the AI agent's processing pipeline.

**Nodes Involved:**  
- When chat message received

**Node Details:**

- **When chat message received**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Webhook trigger node for incoming chat messages, starting the workflow.  
  - Configuration: Uses a webhook ID to expose an endpoint for chat inputs. No extra options configured.  
  - Inputs: External HTTP chat messages  
  - Outputs: Connects directly to AI Agent node’s main input  
  - Version: 1.3  
  - Edge Cases: Webhook misconfiguration, unauthorized calls, network errors, malformed payloads.  
  - Notes: Entry point for the entire conversation-driven workflow.

---

#### 1.2 AI Agent Setup

**Overview:**  
Defines the AI agent’s core logic, instructing it to use multiple thinking tools iteratively and manage the reasoning process.

**Nodes Involved:**  
- AI Agent  
- Simple Memory  
- OpenAI Chat Model

**Node Details:**

- **AI Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Main AI orchestrator that processes input, calls thinking tools, and produces responses.  
  - Configuration: System message instructs the agent to start by calling “Initial thoughts” tool, then “Additional thoughts” after executing initial plans, guiding multi-step reasoning with tools X, Y, and Z (placeholders).  
  - Inputs: Receives chat messages and connects to memory, language model, and thinking tools.  
  - Outputs: Produces final AI-generated responses.  
  - Version: 2.2  
  - Edge Cases: Misconfigured prompts may confuse agent behavior; failure in tool invocation; rate limits or API errors from language model.  
  - Notes: Core logic node for multi-step reasoning AI.

- **Simple Memory**  
  - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - Role: Maintains conversation context in a sliding window buffer to provide to the AI agent.  
  - Configuration: Default buffer window (no parameters customized).  
  - Inputs: Connected from AI Agent’s memory output.  
  - Outputs: Supplies memory context back to AI Agent.  
  - Version: 1.3  
  - Edge Cases: Memory overflow or loss of relevant context if window size is too small.

- **OpenAI Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: The underlying language model (GPT-4) powering the AI agent's responses.  
  - Configuration: Model set to "gpt-4.1-mini" variant; OAuth credentials configured (named "Duv's OpenAI").  
  - Inputs: Connected to AI Agent as the language model provider.  
  - Outputs: Returns generated text to AI Agent.  
  - Version: 1.2  
  - Edge Cases: API quota or authentication errors, model unavailability, network latency.

---

#### 1.3 Thinking Tools

**Overview:**  
Two specialized tools implemented as calls to a reusable sub-workflow, enabling multi-step reasoning by the agent: one for initial planning (“Initial thoughts”) and another for reflection and further planning (“Additional thoughts”).

**Nodes Involved:**  
- Initial thoughts  
- Additional thoughts

**Node Details:**

- **Initial thoughts**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Tool node calling a sub-workflow to generate initial concise thoughts and planning for the user query.  
  - Configuration:  
    - Calls sub-workflow with ID "KNxyzmWuqSCK1GUR" (named "TEMPLATE - AI agent with multiple thinking tools").  
    - Passes a prompt requesting concise initial thoughts in a token-efficient list format.  
    - Defines workflow input variable “Thoughts” populated via an AI-generated expression.  
  - Inputs: Invoked by AI Agent as first tool to use.  
  - Outputs: Returns initial thoughts text to AI Agent.  
  - Version: 2.2  
  - Edge Cases: Sub-workflow failures, expression evaluation errors, invalid or empty thoughts output.

- **Additional thoughts**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Similar to Initial thoughts but called after initial planning to reflect on results and decide if further tool usage is required before answering.  
  - Configuration:  
    - Calls same sub-workflow as Initial thoughts.  
    - Passes a prompt requesting concise thoughts focusing on reflection and next steps.  
    - Uses same input/output schema as Initial thoughts.  
  - Inputs: Invoked by AI Agent as subsequent reasoning step.  
  - Outputs: Returns reflective thoughts to AI Agent.  
  - Version: 2.2  
  - Edge Cases: Same sub-workflow and expression risks as Initial thoughts.

---

#### 1.4 AI Language Model & Memory

**Overview:**  
Provides the foundational language model and memory buffer nodes required by the AI agent.

**Nodes Involved:**  
- OpenAI Chat Model  
- Simple Memory

**Node Details:**  
Covered in 1.2 AI Agent Setup for details.

---

#### 1.5 Sub-Workflow Execution

**Overview:**  
Sub-workflow node acting as the reusable thinking tool implementation that accepts a “Thoughts” string input and returns a concise list of thoughts. This workflow is invoked twice, enabling multi-step reasoning.

**Nodes Involved:**  
- Thinking sub-workflow (executeWorkflowTrigger node)

**Node Details:**

- **Thinking sub-workflow**  
  - Type: `n8n-nodes-base.executeWorkflowTrigger`  
  - Role: Trigger node for the reusable sub-workflow implementing the thinking tool.  
  - Configuration: Accepts input parameter “Thoughts” (string).  
  - Inputs: Receives the “Thoughts” prompt from calling tool nodes.  
  - Outputs: Returns generated thoughts to calling tool node.  
  - Version: 1.1  
  - Edge Cases: Sub-workflow execution errors, input mismatches, or missing parameters.  
  - Notes: Acts as a "scratchpad" for the AI agent’s internal reasoning steps.

---

#### 1.6 Documentation & Instructional Notes

**Overview:**  
Sticky notes placed in the canvas explain the workflow’s purpose, usage instructions, and customization tips.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3

**Node Details:**

- **Sticky Note** (positioned left, large)  
  - Content explains the advanced agent reasoning framework, how the template works, and how to customize it for new tools and use cases.  
  - Serves to guide users on adapting the template.

- **Sticky Note1** (near thinking tools)  
  - Notes that users can adjust prompts of thinking tools and add other tools as needed.

- **Sticky Note2** (near sub-workflow)  
  - Describes the sub-workflow as a mimic of the thinking tool, accepting a “Thought” string input.

- **Sticky Note3** (near AI Agent)  
  - Reminds users to customize the AI Agent's system prompt for their use case.

---

### 3. Summary Table

| Node Name               | Node Type                                    | Functional Role                        | Input Node(s)               | Output Node(s)          | Sticky Note                                                                                                                            |
|-------------------------|----------------------------------------------|-------------------------------------|-----------------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger         | Receives chat messages to start workflow | -                           | AI Agent                | # Advanced Agent Reasoning Framework ... (see Sticky Note content in Node 1.6)                                                         |
| AI Agent                | @n8n/n8n-nodes-langchain.agent               | Orchestrates AI multi-step reasoning  | When chat message received, Simple Memory, OpenAI Chat Model, Initial thoughts, Additional thoughts | -                       | ## The agent - customize system prompt (Sticky Note3)                                                                                   |
| Simple Memory           | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores conversation memory context   | AI Agent                    | AI Agent                |                                                                                                                                         |
| OpenAI Chat Model       | @n8n/n8n-nodes-langchain.lmChatOpenAi        | GPT-4 language model provider         | AI Agent                    | AI Agent                |                                                                                                                                         |
| Initial thoughts        | @n8n/n8n-nodes-langchain.toolWorkflow        | First thinking tool: initial planning | AI Agent                    | AI Agent                | ## Your thinking tools - adjust prompts and add tools as needed (Sticky Note1)                                                          |
| Additional thoughts     | @n8n/n8n-nodes-langchain.toolWorkflow        | Second thinking tool: reflection/planning | AI Agent                    | AI Agent                | ## Your thinking tools - adjust prompts and add tools as needed (Sticky Note1)                                                          |
| Thinking sub-workflow   | n8n-nodes-base.executeWorkflowTrigger        | Sub-workflow implementing thinking tool logic | Called by Initial thoughts, Additional thoughts | Returns thoughts to calling tool | ## The subworkflow - accepts "Thought" string input (Sticky Note2)                                                                       |
| Sticky Note             | n8n-nodes-base.stickyNote                     | Documentation and instructions        | -                           | -                       | See content in section 1.6                                                                                                              |
| Sticky Note1            | n8n-nodes-base.stickyNote                     | Notes on thinking tools               | -                           | -                       | See content in section 1.6                                                                                                              |
| Sticky Note2            | n8n-nodes-base.stickyNote                     | Notes on thinking sub-workflow        | -                           | -                       | See content in section 1.6                                                                                                              |
| Sticky Note3            | n8n-nodes-base.stickyNote                     | Notes on AI Agent prompt customization | -                           | -                       | See content in section 1.6                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Trigger Node**  
   - Add node: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: "When chat message received"  
   - Configure webhook ID (auto-generated or custom)  
   - Leave options default  
   - Position: top-left

2. **Add AI Agent Node**  
   - Add node: `@n8n/n8n-nodes-langchain.agent`  
   - Name: "AI Agent"  
   - System Message:  
     ```
     You are a very smart assistant.

     You always start by calling the tool "Initial thoughts" to plan the way you'll proceed to use the tools X, Y, and Z.

     Once you've executed your initial plan, call the tool "Additional thoughts" to check in with the results and decide if you need to further use tools X, Y, and Z or if you're ready to answer the user.
     ```  
   - Version: 2.2  
   - Position: center-left

3. **Add OpenAI Chat Model Node**  
   - Add node: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: "OpenAI Chat Model"  
   - Model: select "gpt-4.1-mini" from model list  
   - Credentials: Link an OpenAI API credential (e.g., OAuth2 or API key)  
   - Version: 1.2  
   - Position: below AI Agent

4. **Add Simple Memory Node**  
   - Add node: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Name: "Simple Memory"  
   - Use default settings (sliding window buffer)  
   - Version: 1.3  
   - Position: right of OpenAI Chat Model

5. **Connect Nodes for AI Agent Setup**  
   - Connect "When chat message received" → AI Agent (main input)  
   - Connect "Simple Memory" → AI Agent (ai_memory input)  
   - Connect "OpenAI Chat Model" → AI Agent (ai_languageModel input)

6. **Create Thinking Sub-Workflow**  
   - Create a new workflow named "TEMPLATE - AI agent with multiple thinking tools" (or any name)  
   - Add an `executeWorkflowTrigger` node at the start  
   - Configure input variable: name "Thoughts" (string)  
   - This sub-workflow should implement the logic to generate concise thoughts based on input (e.g., an OpenAI node generating text from "Thoughts" prompt)  
   - Save and note the workflow ID

7. **Add "Initial thoughts" Tool Node**  
   - Add node: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Name: "Initial thoughts"  
   - Set workflowId to the sub-workflow created in step 6  
   - Description: "Always start your process by calling this tool to write initial thoughts and plan the way you'll go about answering the user query."  
   - Map input "Thoughts" to an AI expression generating concise initial thoughts (e.g., `$fromAI('Thoughts', 'Write initial thoughts very concisely (be token efficient, just list some thoughts) on the best ways to go about the user query.', 'string')`)  
   - Version: 2.2  
   - Position: right of AI Agent

8. **Add "Additional thoughts" Tool Node**  
   - Add node: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Name: "Additional thoughts"  
   - Use same workflowId as "Initial thoughts" node  
   - Description: "Call this tool after having ... to check-in and decide if other steps would be needed before being able to finally answer the user query."  
   - Map input "Thoughts" to AI-generated concise reflective thoughts (similar expression but prompt adjusted accordingly)  
   - Version: 2.2  
   - Position: right of "Initial thoughts"

9. **Connect Thinking Tools to AI Agent**  
   - Connect "Initial thoughts" → AI Agent (ai_tool input index 0)  
   - Connect "Additional thoughts" → AI Agent (ai_tool input index 0)

10. **Final Connections and Activate**  
    - Validate all connections and configurations  
    - Activate the workflow  
    - Test by sending chat messages to the webhook endpoint

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This template demonstrates how to give an AI agent multiple custom "thinking" steps to build more powerful and reliable automations, bypassing the single "Think Tool" limit by using a reusable sub-workflow.                 | Sticky Note at workflow start                                                                       |
| The sub-workflow imitates the thinking tool, accepting a string called "Thought" as input.                                                                                                                                  | Sticky Note near “Thinking sub-workflow” node                                                      |
| Feel free to adjust the prompts of the thinking tools, add more tools, and connect other necessary tools to customize the workflow according to your needs.                                                                  | Sticky Note near thinking tools                                                                    |
| Don't forget to customize the AI Agent's system prompt to your use case to guide the multi-step reasoning process effectively.                                                                                              | Sticky Note near AI Agent node                                                                     |
| For more advanced customization and examples, visit the official n8n documentation on LangChain nodes and AI agent pipelines: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                                  | Official n8n LangChain documentation                                                               |
| OpenAI API credentials must be set up correctly with appropriate permissions and quota to avoid authentication or rate limit errors.                                                                                        | Credential setup instructions for OpenAI                                                          |

---

**Disclaimer:**  
The provided text is extracted exclusively from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.