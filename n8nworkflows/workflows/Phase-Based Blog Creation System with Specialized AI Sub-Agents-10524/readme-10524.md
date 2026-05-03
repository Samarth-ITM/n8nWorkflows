Phase-Based Blog Creation System with Specialized AI Sub-Agents

https://n8nworkflows.xyz/workflows/phase-based-blog-creation-system-with-specialized-ai-sub-agents-10524


# Phase-Based Blog Creation System with Specialized AI Sub-Agents

### 1. Workflow Overview

This workflow, titled **"Phase-Based Blog Creation System with Specialized AI Sub-Agents"**, orchestrates a multi-step AI-driven content creation process tailored for producing well-structured blog posts. It is designed primarily for content teams, marketing agencies, or automated content generation systems requiring modular, phase-based writing with specialized AI agents for each writing phase.

The workflow logically divides into these functional blocks:

- **1.1 Input Reception & Orchestration:** Captures incoming content requests via chat or Slack triggers and routes tasks to specialized AI agents.
- **1.2 Memory Management:** Maintains conversational or contextual memory to provide continuity across AI agent interactions.
- **1.3 AI Sub-Agent Execution:** Calls specialized AI sub-workflows (agents) dedicated to discrete blog writing phases:
  - Headline writing
  - Hook writing
  - Introduction writing
  - Outline creation
  - Research gathering (optional)
  - Blog body writing
  - Source and quotes extraction
  - Final draft editing
- **1.4 Optional Research Integration:** Disabled nodes hint at external research integration and data storage.
- **1.5 Output Handling:** Manages the final output flow, including Slack response and output return (some disabled).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Orchestration

**Overview:**  
This block receives input from chat or Slack triggers, feeds it into the main AI orchestrator node, and directs the process flow toward writing sub-agents. It serves as the entry point and decision hub.

**Nodes Involved:**  
- Chat (chatTrigger)  
- Slack Message Received (slackTrigger) [disabled]  
- Orchestrator (langchain.agent)  
- Slack Message Response (slack) [disabled]

**Node Details:**

- **Chat**  
  - *Type:* Chat trigger node  
  - *Role:* Receives chat input from an external chatbot or UI  
  - *Config:* Webhook with ID "670baf29-2ec7-43a5-b781-e434a7deb7c5"  
  - *Connections:* Outputs to Orchestrator  
  - *Potential Failures:* Webhook misconfiguration, timeout, invalid input format  

- **Slack Message Received [disabled]**  
  - *Type:* Slack trigger node  
  - *Role:* Alternative input source from Slack messages  
  - *Note:* Disabled, indicating Slack input is optional or deprecated  
  - *Potential Failures:* Slack API auth errors, missing scopes  

- **Orchestrator**  
  - *Type:* LangChain Agent node  
  - *Role:* Central AI orchestrator that routes tasks to specialized sub-agents/tools based on input  
  - *Config:* No explicit parameters, uses configured AI language model (Claude) and memory  
  - *Connections:*  
    - Receives input from Chat and Slack Message Received  
    - Sends output to multiple toolWorkflow nodes (sub-agents)  
    - Connected to Simple Memory and Claude nodes for context and language model  
  - *Expressions:* Uses AI memory and tool interfaces  
  - *Edge Cases:* AI model timeout, routing failures, memory overflow  

- **Slack Message Response [disabled]**  
  - *Type:* Slack output node  
  - *Role:* Sends response back to Slack channel  
  - *Disabled:* Indicates Slack response not currently active  

---

#### 2.2 Memory Management

**Overview:**  
Preserves conversational or contextual memory for the orchestrator to maintain state across interactions, improving continuity and coherence in AI responses.

**Nodes Involved:**  
- Simple Memory (langchain.memoryBufferWindow)

**Node Details:**

- **Simple Memory**  
  - *Type:* LangChain Memory node  
  - *Role:* Maintains a sliding window of conversation history or relevant context  
  - *Config:* Default buffer window memory with no custom parameters shown  
  - *Connections:* Feeds memory into Orchestrator node  
  - *Edge Cases:* Memory size limits, data loss on restart, memory overflow  

---

#### 2.3 AI Sub-Agent Execution

**Overview:**  
Executes specialized AI sub-agents implemented as separate tool workflows, each responsible for a specific part of the blog creation process. These modular sub-agents enable focused content generation phases.

**Nodes Involved:**  
- Call Headline Writer Agent (toolWorkflow)  
- Call Hook Writer Agent (toolWorkflow)  
- Call Intro Writer Agent (toolWorkflow)  
- Call Outline Writer Agent (toolWorkflow)  
- Call Blog Writer Agent (toolWorkflow)  
- Call Sources and Quotes Agent (toolWorkflow)  
- Call Final Draft Editor Agent (toolWorkflow)  
- Do Research (toolWorkflow) [optional research]  
- Headlines Writer (langchain.agent) [disabled]  
- Headlines Writer (Tool) (langchain.agentTool) [disabled]

**Node Details:**

- **Call Headline Writer Agent**  
  - *Type:* Tool Workflow node  
  - *Role:* Invokes sub-workflow specialized in generating blog headlines  
  - *Config:* References an external workflow (not included here)  
  - *Input:* From Orchestrator AI tool output  
  - *Output:* Returns headline text for workflow use  
  - *Edge Cases:* Sub-workflow failure, communication timeout  

- **Call Hook Writer Agent**  
  - *Type:* Tool Workflow node  
  - *Role:* Generates engaging hooks or opening sentences  
  - *Same configuration and edge cases as above*  

- **Call Intro Writer Agent**  
  - *Type:* Tool Workflow node  
  - *Role:* Writes the blog introduction paragraph(s)  
  - *Same configuration and edge cases as above*  

- **Call Outline Writer Agent**  
  - *Type:* Tool Workflow node  
  - *Role:* Produces the blog post outline or structure  
  - *Same configuration and edge cases as above*  

- **Call Blog Writer Agent**  
  - *Type:* Tool Workflow node  
  - *Role:* Generates main blog content based on outline and inputs  
  - *Same configuration and edge cases as above*  

- **Call Sources and Quotes Agent**  
  - *Type:* Tool Workflow node  
  - *Role:* Extracts or generates relevant sources and quotes for the blog  
  - *Same configuration and edge cases as above*  

- **Call Final Draft Editor Agent**  
  - *Type:* Tool Workflow node  
  - *Role:* Performs final editorial review and refinement  
  - *Same configuration and edge cases as above*  

- **Do Research [optional]**  
  - *Type:* Tool Workflow node  
  - *Role:* Performs research to gather data or context  
  - *Disabled:* Indicates research is optional or experimental  

- **Headlines Writer (Agent) [disabled]**  
  - *Type:* LangChain Agent node  
  - *Role:* Possibly an older or alternate headline writing method  

- **Headlines Writer (Tool) [disabled]**  
  - *Type:* LangChain Agent Tool node  
  - *Role:* Alternate headline generation tool, currently unused  

---

#### 2.4 Research & Data Storage (Disabled/Optional)

**Overview:**  
Includes nodes that perform HTTP research queries and save results into Google Sheets for persistence or audit. These nodes are disabled, suggesting optional or experimental state.

**Nodes Involved:**  
- Research w/ Perplexity (httpRequest) [disabled]  
- Save Research (googleSheets) [disabled]  
- Get Research (googleSheetsTool)  
- Return Output (noOp) [disabled]  
- When Executed by Another Workflow (executeWorkflowTrigger) [disabled]

**Node Details:**

- **Research w/ Perplexity**  
  - *Type:* HTTP Request node  
  - *Role:* Calls Perplexity API or similar for external research  
  - *Disabled:* Not active in current workflow  

- **Save Research**  
  - *Type:* Google Sheets node  
  - *Role:* Persists research data into a spreadsheet  
  - *Disabled*  

- **Get Research**  
  - *Type:* Google Sheets Tool node  
  - *Role:* Reads stored research data for use in AI process  
  - *Active*  

- **Return Output**  
  - *Type:* No Operation node  
  - *Role:* Placeholder for final output step  
  - *Disabled*  

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point when called as sub-workflow (disabled)  

---

#### 2.5 Language Model Configuration

**Overview:**  
Defines the AI language model used for content generation, connected to the orchestrator.

**Nodes Involved:**  
- Claude (langchain.lmChatAnthropic)

**Node Details:**

- **Claude**  
  - *Type:* LangChain Anthropic Chat model node  
  - *Role:* Provides the underlying AI language model for Orchestrator  
  - *Config:* Default Anthropic Claude model, no parameters shown  
  - *Connections:* Feeds into Orchestrator’s ai_languageModel input  
  - *Potential Failures:* API key issues, rate limits, model downtime  

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                                | Input Node(s)            | Output Node(s)                | Sticky Note                              |
|-----------------------------|----------------------------------|------------------------------------------------|--------------------------|------------------------------|-----------------------------------------|
| Chat                        | chatTrigger                      | Entry point: receives chat input               | -                        | Orchestrator                 |                                         |
| Slack Message Received       | slackTrigger (disabled)          | Entry point via Slack (disabled)                | -                        | Orchestrator                 |                                         |
| Orchestrator                | langchain.agent                  | Central AI orchestrator routing to sub-agents | Chat, Slack Message Received | Call Headline Writer Agent, Call Hook Writer Agent, Call Intro Writer Agent, Call Outline Writer Agent, Call Blog Writer Agent, Call Sources and Quotes Agent, Call Final Draft Editor Agent, Do Research, Get Research, Slack Message Response |                                         |
| Simple Memory               | langchain.memoryBufferWindow     | Maintains conversation/context memory          | -                        | Orchestrator                 |                                         |
| Call Headline Writer Agent  | langchain.toolWorkflow           | Sub-agent: headline generation                  | Orchestrator             | -                            |                                         |
| Call Hook Writer Agent      | langchain.toolWorkflow           | Sub-agent: hook writing                          | Orchestrator             | -                            |                                         |
| Call Intro Writer Agent     | langchain.toolWorkflow           | Sub-agent: introduction writing                  | Orchestrator             | -                            |                                         |
| Call Outline Writer Agent   | langchain.toolWorkflow           | Sub-agent: outline creation                      | Orchestrator             | -                            |                                         |
| Call Blog Writer Agent      | langchain.toolWorkflow           | Sub-agent: main blog content generation          | Orchestrator             | -                            |                                         |
| Call Sources and Quotes Agent | langchain.toolWorkflow         | Sub-agent: sources and quotes extraction         | Orchestrator             | -                            |                                         |
| Call Final Draft Editor Agent | langchain.toolWorkflow         | Sub-agent: final content editing                  | Orchestrator             | -                            |                                         |
| Do Research                 | langchain.toolWorkflow (disabled) | Optional research sub-agent                      | Orchestrator             | -                            |                                         |
| Research w/ Perplexity      | httpRequest (disabled)           | External research HTTP API call                  | -                        | Save Research                |                                         |
| Save Research               | googleSheets (disabled)          | Persists research data                           | Research w/ Perplexity   | Return Output                |                                         |
| Get Research                | googleSheetsTool                 | Retrieves research data                           | -                        | Orchestrator                 |                                         |
| Return Output               | noOp (disabled)                 | Placeholder for output                           | Save Research            | -                            |                                         |
| When Executed by Another Workflow | executeWorkflowTrigger (disabled) | Sub-workflow entry point (disabled)         | -                        | Headlines Writer             |                                         |
| Headlines Writer            | langchain.agent (disabled)       | Alternate headline agent                         | When Executed by Another Workflow | -                     |                                         |
| Headlines Writer (Tool)     | langchain.agentTool (disabled)   | Alternate headline generation tool              | -                        | Orchestrator                 |                                         |
| Slack Message Response      | slack (disabled)                | Sends Slack reply                               | Orchestrator             | -                            |                                         |
| Claude                      | langchain.lmChatAnthropic       | AI Language Model                               | -                        | Orchestrator                 |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node:**  
   - Type: Chat Trigger  
   - Configure webhook to receive chat input (set webhook ID)  
   - No special parameters needed  

2. **Create Orchestrator Node:**  
   - Type: LangChain Agent  
   - Connect Chat Trigger output to Orchestrator input  
   - Configure AI language model input (connect to Claude node)  
   - Configure AI memory input (connect to Simple Memory node)  
   - Configure AI tool input to connect to sub-agent tool workflows  

3. **Create Simple Memory Node:**  
   - Type: LangChain Memory Buffer Window  
   - Default settings for conversation memory buffer  
   - Connect output to Orchestrator’s memory input  

4. **Create Claude Node:**  
   - Type: LangChain Anthropic Chat Model  
   - Configure API credentials for Anthropic Claude model  
   - Connect output to Orchestrator’s AI model input  

5. **Create Sub-Agent Tool Workflow Nodes:**  
   For each blog writing phase, add a Tool Workflow node:  
   - Call Headline Writer Agent  
   - Call Hook Writer Agent  
   - Call Intro Writer Agent  
   - Call Outline Writer Agent  
   - Call Blog Writer Agent  
   - Call Sources and Quotes Agent  
   - Call Final Draft Editor Agent  
   - Optionally, Do Research (disabled by default)  

   Configure each with the corresponding workflow ID or path for the specialized sub-agent. Connect their inputs to Orchestrator’s AI tool output.  

6. **(Optional) Create Research & Data Storage Nodes:**  
   - HTTP Request node for external research API (Perplexity)  
   - Google Sheets node to save research  
   - Google Sheets Tool to retrieve research  
   - Disabled by default. Configure credentials and endpoints if enabled.  

7. **Create Slack Message Response Node (optional):**  
   - Type: Slack node  
   - Configure Slack OAuth2 credentials and channel  
   - Connect output of Orchestrator to Slack Message Response node  

8. **Wire all nodes:**  
   - Chat Trigger → Orchestrator  
   - Claude → Orchestrator (ai_languageModel)  
   - Simple Memory → Orchestrator (ai_memory)  
   - Orchestrator (ai_tool) → all Call *Writer Agent nodes  
   - Call *Writer Agent nodes execute sub-workflows and return to Orchestrator  
   - Orchestrator → Slack Message Response (optional)  

9. **Set Credentials:**  
   - Configure Anthropic Claude API credentials for Claude node  
   - Configure Slack OAuth2 for Slack nodes if used  
   - Configure Google Sheets OAuth2 if research nodes enabled  
   - Configure any required API keys for research HTTP request  

10. **Test and debug:**  
    - Trigger Chat node with sample input  
    - Verify orchestrator routes correctly and calls sub-agents  
    - Check memory buffer continuity  
    - Validate output generation and Slack response if enabled  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                  |
|--------------------------------------------------------------------------------------------------------------|---------------------------------|
| The workflow uses Anthropic Claude as the primary language model.                                            | AI language model configuration |
| Slack integration nodes are present but currently disabled, indicating optional Slack messaging support.    | Slack API & Slack OAuth2 setup  |
| Multiple sticky notes are present in the workflow JSON but contain no content, implying placeholders or documentation areas. | Workflow design notes           |
| Workflow uses n8n LangChain nodes extensively for modular AI orchestration and memory management.           | n8n LangChain documentation     |
| Disabled research nodes suggest extensibility for integrating external data sources like Perplexity API.    | External research integration   |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.