Create Custom Reasoning Patterns for AI Agents with GraphRAG & Knowledge Ontology

https://n8nworkflows.xyz/workflows/create-custom-reasoning-patterns-for-ai-agents-with-graphrag---knowledge-ontology-6816


# Create Custom Reasoning Patterns for AI Agents with GraphRAG & Knowledge Ontology

---
### 1. Workflow Overview

This workflow, titled **"Reasoning Expert with Graph RAG Knowledge Ontology"**, is designed to create and leverage custom reasoning patterns for AI agents using a combination of an AI language model, interaction dynamics expertise, and a knowledge graph ontology. It integrates LangChain nodes with an external expert API (InfraNodus) to enhance AI responses by contextualizing user queries within a structured reasoning ontology.

**Target Use Cases:**  
- Advanced chatbot implementations requiring nuanced reasoning and adaptive response generation  
- AI agents that improve their interactions based on dynamic advice from an expert system analyzing interaction dynamics  
- Leveraging knowledge graphs to guide AI reasoning and enrich conversational quality  

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving chat messages to trigger the workflow  
- **1.2 Reasoning Agent Configuration:** Defining the AI agent that processes input and integrates external expert advice  
- **1.3 AI Language Model and Memory:** Utilizing OpenAI GPT-4o-mini model with conversational memory buffer  
- **1.4 Interaction Dynamics Expert API Call:** Querying an external expert system for dynamic interaction advice based on the ongoing conversation  
- **1.5 Knowledge Ontology Context:** Using an InfraNodus knowledge graph as the reasoning ontology to augment the AI agent’s understanding  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming chat messages and triggers the workflow to process each user query.

**Nodes Involved:**  
- When chat message received

**Node Details:**  
- **Name:** When chat message received  
- **Type:** LangChain Chat Trigger  
- **Technical Role:** Webhook node that activates the workflow upon receiving a chat message  
- **Configuration:** Default options, no special filters or conditions  
- **Key Expressions:** None; raw incoming message triggers the flow  
- **Input Connections:** External webhook call (chat message)  
- **Output Connections:** Feeds into the Reasoning Agent node  
- **Version Requirements:** Type version 1.1  
- **Failure Modes:** Possible webhook misconfiguration, network issues preventing message receipt  

---

#### 1.2 Reasoning Agent Configuration

**Overview:**  
Defines a reasoning agent that acts as the main AI processor. It interprets conversation context, queries the expert system, and augments the response based on expert advice.

**Nodes Involved:**  
- Reasoning Agent  
- Sticky Note (explaining system prompt)

**Node Details:**  
- **Name:** Reasoning Agent  
- **Type:** LangChain Agent Node  
- **Technical Role:** Core AI agent combining language model, memory, and tools to generate responses  
- **Configuration:**  
  - System message instructs the agent to utilize a dynamic interaction expert’s advice to improve responses  
  - Emphasizes importance of expert’s feedback to augment the original prompt  
- **Key Expressions:** Static system prompt text configured directly in parameters  
- **Input Connections:** Receives chat messages from "When chat message received"  
- **Output Connections:** Not connected further in this workflow (could be output to chat)  
- **Version Requirements:** Type version 1.9  
- **Failure Modes:** Expression or prompt parsing errors, misinterpretation of expert advice, agent timeout  
- **Sticky Note:** Describes the system prompt’s purpose to augment user queries with reasoning ontology knowledge  

---

#### 1.3 AI Language Model and Memory

**Overview:**  
Provides the AI agent with a language model backend (OpenAI GPT-4o-mini) and conversation memory to maintain context over interactions.

**Nodes Involved:**  
- OpenAI Chat Model  
- Simple Memory  

**Node Details:**  
- **OpenAI Chat Model:**  
  - Type: LangChain OpenAI Chat Model Node  
  - Role: Generates language responses using GPT-4o-mini  
  - Configuration: Model set to "gpt-4o-mini"; uses OpenAI API credentials  
  - Inputs: Connected into the Reasoning Agent as the language model  
  - Outputs: To Reasoning Agent node  
  - Version: 1.2  
  - Failure Modes: API errors, rate limits, auth failures  
- **Simple Memory:**  
  - Type: LangChain Buffer Window Memory  
  - Role: Maintains a sliding window of conversational context for the agent  
  - Configuration: Default buffer window, no custom params  
  - Inputs: Connected into the Reasoning Agent as AI memory source  
  - Outputs: To Reasoning Agent node  
  - Version: 1.3  
  - Failure Modes: Memory overflow, expression errors  

---

#### 1.4 Interaction Dynamics Expert API Call

**Overview:**  
Invokes an external HTTP API (InfraNodus) that serves as a reasoning expert specializing in interaction dynamics. It analyzes the conversation state and provides advice to improve the agent's responses.

**Nodes Involved:**  
- Interaction Dynamics Expert  

**Node Details:**  
- **Name:** Interaction Dynamics Expert  
- **Type:** HTTP Request Tool  
- **Technical Role:** Sends POST requests to InfraNodus API with user query and interaction context, receives expert advice  
- **Configuration:**  
  - URL: InfraNodus expert API endpoint with query parameters controlling data returned  
  - Method: POST  
  - Authentication: HTTP Bearer token (InfraNodus Experts Account credential)  
  - Body Parameters: Includes fixed name "eightos_system", requestMode "response", prompt dynamically set by AI expression from the Reasoning Agent, and aiTopics flag true  
  - Tool Description: Contains detailed domain concepts, topics, and relations that define the reasoning ontology context for the expert  
- **Key Expressions:** Uses a special `$fromAI()` expression to dynamically inject the user query interpretation from AI input parameters  
- **Input Connections:** Feeds into the Reasoning Agent as an AI tool  
- **Outputs:** Response passed back to Reasoning Agent for response augmentation  
- **Version:** 4.2  
- **Failure Modes:** API downtime, auth token expiry, malformed requests, network timeouts  

---

#### 1.5 Knowledge Ontology Context

**Overview:**  
Provides documentation and context for the knowledge graph ontology used to support reasoning patterns. This block consists solely of sticky notes as guidance and informational content for maintainers.

**Nodes Involved:**  
- Sticky Note1  
- Sticky Note2  

**Node Details:**  
- **Sticky Note1:**  
  - Content: Explains the use of InfraNodus knowledge graph as reasoning ontology, instructions to specify graph name and description  
- **Sticky Note2:**  
  - Content: Guides on auto-generating and manually editing the ontology graph, includes links to InfraNodus AI ontologies interface and support portal  
  - Embedded image link: InfraNodus knowledge graph example  
- **Role:** Informational only; no input/output connections or processing  
- **Failure Modes:** None (documentation only)  

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                   | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                                 |
|-------------------------------|----------------------------------|---------------------------------|-----------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------|
| When chat message received     | @n8n/n8n-nodes-langchain.chatTrigger | Input reception trigger          | External webhook             | Reasoning Agent           |                                                                                                             |
| Reasoning Agent                | @n8n/n8n-nodes-langchain.agent   | Central reasoning AI agent       | When chat message received, OpenAI Chat Model, Simple Memory, Interaction Dynamics Expert | None                     | ## 1. Reasoning Agent: System prompt augments original prompt using reasoning ontology knowledge             |
| OpenAI Chat Model             | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model backend           | None                        | Reasoning Agent           |                                                                                                             |
| Simple Memory                 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversational memory buffer     | None                        | Reasoning Agent           |                                                                                                             |
| Interaction Dynamics Expert   | n8n-nodes-base.httpRequestTool   | External expert API tool         | None                        | Reasoning Agent           |                                                                                                             |
| Sticky Note                   | n8n-nodes-base.stickyNote        | Documentation: Reasoning Agent   | None                        | None                     | ## 1. Reasoning Agent: Here you add a system prompt that tells the agent to augment the original prompt       |
| Sticky Note1                  | n8n-nodes-base.stickyNote        | Documentation: Reasoning Ontology | None                        | None                     | ## 2. Reasoning Ontology: Add InfraNodus knowledge graph with name, description, and topical summary          |
| Sticky Note2                  | n8n-nodes-base.stickyNote        | Documentation: Knowledge Graph   | None                        | None                     | Knowledge graph can be auto-generated or edited manually; includes links to InfraNodus AI ontologies support |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Incoming Chat Trigger Node**  
   - Add node: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name it: `When chat message received`  
   - Leave default parameters (listening for any chat message)  
   - This node creates a webhook to start the workflow on chat input  

2. **Create the Reasoning Agent Node**  
   - Add node: `@n8n/n8n-nodes-langchain.agent`  
   - Name it: `Reasoning Agent`  
   - Configure system message with the following prompt:  
     ```
     You are a reasoning agent. You have access to a dynamic interaction expert that provides you advice on how to continue your interaction. When you send a request to this expert, you need to give it an interpretation of the previous interaction and its dynamics (your interpretation of the conversation) using the language and concepts that the reasoning agent will understand. 

     Use the response from the expert as an instruction to improve your response to the user's query. Give the utmost importance to the expert's advice to improve your standard response to the client's original query.
     ```  
   - This node will orchestrate the reasoning process  
   - Connect input from `When chat message received` node  

3. **Add OpenAI Chat Model Node**  
   - Add node: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name it: `OpenAI Chat Model`  
   - Set model to `gpt-4o-mini`  
   - Configure OpenAI API credentials (create or select existing valid OpenAI API credential)  
   - Connect output to `Reasoning Agent` under AI language model input  

4. **Add Simple Memory Node**  
   - Add node: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Name it: `Simple Memory`  
   - Use default parameters (sliding window buffer for chat context)  
   - Connect output to `Reasoning Agent` under AI memory input  

5. **Add Interaction Dynamics Expert API HTTP Request Node**  
   - Add node: `n8n-nodes-base.httpRequestTool`  
   - Name it: `Interaction Dynamics Expert`  
   - Set method to `POST`  
   - URL: `https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&addStats=true&optimize=develop&includeStatements=true&includeGraphSummary=true&includeGraph=false`  
   - Authentication Type: HTTP Bearer Token  
   - Configure credentials for InfraNodus Experts Account (or equivalent valid token)  
   - Body Parameters (JSON):  
     ```json
     {
       "name": "eightos_system",
       "requestMode": "response",
       "prompt": "={{ $fromAI('parameters2_Value', `User query to send to the expert`, 'string') }}",
       "aiTopics": "true"
     }
     ```  
   - Connect output to `Reasoning Agent` as AI tool input  

6. **Connect Nodes Properly**  
   - `When chat message received` → `Reasoning Agent` (main input)  
   - `OpenAI Chat Model` → `Reasoning Agent` (ai_languageModel)  
   - `Simple Memory` → `Reasoning Agent` (ai_memory)  
   - `Interaction Dynamics Expert` → `Reasoning Agent` (ai_tool)  

7. **Add Informational Sticky Notes (Optional but Recommended)**  
   - Add sticky notes with content describing the system prompt, the reasoning ontology, and instructions on InfraNodus knowledge graph usage  
   - Position notes next to relevant nodes for clarity  

8. **Workflow Settings**  
   - Execution order: default (v1)  
   - Activate webhook and deploy workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                              |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| InfraNodus knowledge graph can be auto-generated or edited manually using [[wiki links]] and [tags]                  | https://infranodus.com/import/ai-ontologies                                                                                  |
| Learn more about using knowledge graphs as reasoning experts                                                       | https://support.noduslabs.com/hc/en-us/articles/21429518472988-Using-Knowledge-Graphs-as-Reasoning-Experts                     |
| InfraNodus interaction dynamics expert provides advanced advice on complex social or conversational dynamics         | API documentation and credentials required from InfraNodus expert portal                                                      |
| This workflow integrates LangChain AI nodes with external expert tooling for enhanced reasoning in chatbot contexts | n8n LangChain nodes documentation at https://docs.n8n.io/integrations/builtin/nodes/langchain/                                |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly respects current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.