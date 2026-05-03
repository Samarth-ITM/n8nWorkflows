Build a Voice AI Chatbot with ElevenLabs and InfraNodus Knowledge Experts

https://n8nworkflows.xyz/workflows/build-a-voice-ai-chatbot-with-elevenlabs-and-infranodus-knowledge-experts-4484


# Build a Voice AI Chatbot with ElevenLabs and InfraNodus Knowledge Experts

### 1. Workflow Overview

This n8n workflow, titled **"Build a Voice AI Chatbot with ElevenLabs and InfraNodus Knowledge Experts"**, is designed to create an AI-driven voice chatbot that interacts with users via ElevenLabs Conversational AI and leverages multiple expert knowledge bases powered by InfraNodus knowledge graphs. The workflow orchestrates an AI agent that dynamically selects relevant expert knowledge sources based on the user's query, synthesizes responses, and returns them to the user in conversational form.

The typical use cases include advanced conversational agents for support, ideation, and knowledge discovery that require contextual understanding from specialized knowledge domains and conversational memory.

The workflow is logically divided into these main blocks:

- **1.1 Chat Trigger (Input Reception)**: Receives incoming user messages via webhook from the ElevenLabs AI conversational agent.
- **1.2 Chat Memory Management**: Tracks conversation context using memory buffers keyed by session identifiers.
- **1.3 AI Agent Processing (Orchestration & Expert Selection)**: Uses an orchestrating AI agent which routes queries to multiple expert tools (InfraNodus-based knowledge graphs) and synthesizes a combined response.
- **1.4 Expert Knowledge Requests (InfraNodus Experts)**: Calls several InfraNodus HTTP endpoints representing different expert knowledge domains via HTTP Request nodes.
- **1.5 Response Delivery**: Sends the generated response back to the user through the webhook response node.
- **1.6 Configuration & Documentation**: Sticky notes provide setup instructions for ElevenLabs integration, expert descriptions, and general guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Trigger (Input Reception)

**Overview:**  
This block listens for incoming POST requests containing user messages from the ElevenLabs Conversational AI agent. It acts as the entry point for the workflow.

**Nodes Involved:**  
- Webhook

**Node Details:**

- **Webhook**  
  - Type: `Webhook` (HTTP Trigger)  
  - Configuration:  
    - HTTP Method: POST  
    - Path: Unique webhook ID `"171bf9a6-1390-4195-bd6b-ff3df2e27d1c"`  
    - Response Mode: Respond via a dedicated node (Respond to Webhook)  
  - Inputs: External HTTP POST from ElevenLabs agent  
  - Outputs: Triggers the AI Agent node with the webhook payload  
  - Edge Cases:  
    - Invalid or missing webhook URL leads to failure to trigger  
    - Incoming payload format mismatch may cause downstream errors  

- **Sticky Note6** (attached context)  
  - Describes the need to connect this webhook to the ElevenLabs conversational AI agent, including references to setup instructions and best practices.

---

#### 2.2 Chat Memory Management

**Overview:**  
Keeps track of the conversation context using a sliding window memory buffer keyed by the session ID received from the webhook. This allows the AI agent to maintain context across multiple messages in a session.

**Nodes Involved:**  
- Simple Memory

**Node Details:**

- **Simple Memory**  
  - Type: `Memory Buffer Window` (LangChain memory node)  
  - Configuration:  
    - Session Key: Extracted dynamically from webhook payload via expression `{{$json.body.sessionId}}`  
    - Session ID Type: Custom key based on sessionId  
  - Inputs: Receives webhook payload and passes memory context to AI Agent  
  - Outputs: Provides updated memory context to AI Agent node  
  - Edge Cases:  
    - Missing or malformed sessionId disables memory tracking  
    - Memory overflow or excessive length could degrade performance  
  - Version: Uses version 1.3 of the memory node

- **Sticky Note4**  
  - Explains the purpose of chat memory to track conversational context for user references.

---

#### 2.3 AI Agent Processing (Orchestration & Expert Selection)

**Overview:**  
The central orchestrator that receives the user's prompt and decides which expert tools to call (minimum 1, maximum 3). It modifies the query contextually for each expert, collects their responses, synthesizes them, and generates the final answer for the user.

**Nodes Involved:**  
- AI Agent

**Node Details:**

- **AI Agent**  
  - Type: `LangChain Agent`  
  - Configuration:  
    - Input Text: Extracted from webhook body `{{$json.body.prompt}}`  
    - System Message:  
      - Defines the agent as knowledgeable of Dmitry Paranyushkin's books via tools it can access  
      - Instructions to always use at least one tool and synthesize responses  
      - Encourages combining perspectives and highlighting discrepancies  
    - Prompt Type: `define` (custom prompt definition)  
  - Inputs: Receives user prompt and chat memory  
  - Outputs: Passes synthesized response to Respond to Webhook node  
  - Edge Cases:  
    - Failure to call tools or return responses leads to incomplete answers  
    - Misconfiguration of system prompt can cause faulty routing  
  - Version: 1.9 of the agent node

- **Sticky Note5**  
  - Clarifies the AI agent's role in selecting tools and synthesizing responses, emphasizing well-described tool documentation.

---

#### 2.4 Expert Knowledge Requests (InfraNodus Experts)

**Overview:**  
Each expert is represented by an HTTP Request node connecting to a specific InfraNodus knowledge graph API endpoint. The AI agent queries these experts with a contextually adjusted prompt and receives knowledge graph summaries and statements as responses.

**Nodes Involved:**  
- Special Agent's Manual Book Expert (HTTP Request)  
- Waves into Patterns Book Expert (HTTP Request)  
- The Flow and the Notion Book (HTTP Request)  
- The Polysingularity Letters Book (HTTP Request)

**Node Details for Each Expert Node:**

- **Type:** HTTP Request Tool node specialized for knowledge graph queries  
- **Configuration:**  
  - URL: `https://infranodus.com/api/v1/graphAndAdvice` with query parameters to disable saving, include stats, summaries, and statements  
  - Method: POST  
  - Authentication: HTTP Bearer Auth with InfraNodus API keys  
  - Body Parameters:  
    - `name`: The specific graph/expert name (e.g., `"waves_into_patterns"`)  
    - `prompt`: Dynamically generated prompt adjusted by AI agent (`$fromAI` expression)  
    - `requestMode`: `"response"`  
    - `aiTopics`: `"true"`  
  - Tool Description: Each node contains a detailed description of the expert's domain, listing main topics the expert specializes in.  
- **Inputs:** Receives AI Agent's contextually modified prompt  
- **Outputs:** Sends expert response back to AI Agent for synthesis  
- **Edge Cases:**  
  - API authentication failures  
  - Network timeouts  
  - Invalid prompt causing empty or irrelevant responses  
- **Version:** 4.x HTTP Request Tool node

- **Sticky Notes 1, 2, 3, 9**  
  - Provide instructions on adding more experts, including how to specify the graph name in the body and describe the expert's specialization for better AI agent routing.  
  - Include illustrative images for context and branding.

---

#### 2.5 Response Delivery

**Overview:**  
Once the AI agent synthesizes the final response, this block sends the output back to the calling ElevenLabs conversational AI agent via the webhook response.

**Nodes Involved:**  
- Respond to Webhook

**Node Details:**

- **Respond to Webhook**  
  - Type: `Respond to Webhook`  
  - Configuration: Uses default settings to send the AI Agent's output as the HTTP response to the initial webhook trigger  
  - Inputs: Receives synthesized answer from AI Agent  
  - Outputs: Completes the webhook HTTP response cycle  
  - Edge Cases:  
    - Delayed or missing response can cause user timeout on client side  
    - Malformed output could cause client parsing errors

- **Sticky Note8**  
  - Notes the responsibility of this node to send the final answer back to user chat (specifically mentions Telegram but applicable to any webhook client).

---

#### 2.6 Additional Nodes and Notes

- **OpenAI Model (Disabled)**  
  - A GPT-4o OpenAI model node present but disabled, indicating an alternative or fallback LLM option, possibly for higher precision in expert calling.  
  - Credential linked to OpenAI API account.

- **Google Gemini Chat Model**  
  - Active LangChain node using Google Gemini 2.5 Flash Preview model as the main LLM for the AI Agent.  
  - Credential linked to Google Palm API.

- **Sticky Note10**  
  - Provides commentary on model choice: Google Gemini is faster, OpenAI may be more precise in function/expert calling.

- **Sticky Note & Sticky Note7**  
  - Provide comprehensive instructions on setting up the ElevenLabs Conversational AI agent to integrate with this workflow, including system prompt examples, tool naming conventions, and testing guidelines.  
  - Includes links to detailed tutorials and video walkthroughs.

---

### 3. Summary Table

| Node Name                         | Node Type                        | Functional Role                                      | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                                                                                                                                                                  |
|----------------------------------|---------------------------------|-----------------------------------------------------|---------------------|----------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                          | Webhook                         | Receives user messages from ElevenLabs conversational AI agent | (External trigger)   | AI Agent             | Describes webhook as entry point; instructions to connect with ElevenLabs agent (Sticky Note6)                                                                                                                                                |
| AI Agent                         | LangChain Agent                 | Orchestrates expert tool selection, synthesizes response | Webhook, Simple Memory | Respond to Webhook    | Explains agent's role in choosing experts and synthesizing answers (Sticky Note5)                                                                                                                                                            |
| Respond to Webhook               | Respond to Webhook              | Sends final response back to user                    | AI Agent            | (Webhook Response)   | Delivers response to user chat; mentions Telegram example (Sticky Note8)                                                                                                                                                                    |
| Simple Memory                    | LangChain Memory Buffer Window | Tracks conversation context by sessionId             | Webhook             | AI Agent             | Tracks chat memory to maintain context (Sticky Note4)                                                                                                                                                                                      |
| Special Agent's Manual Book Expert | HTTP Request Tool              | InfraNodus expert query about agency, infiltration, strategies | AI Agent            | AI Agent             | Expert #1 description and instructions for adding InfraNodus graph (Sticky Note1)                                                                                                                                                           |
| Waves into Patterns Book Expert  | HTTP Request Tool               | InfraNodus expert for natural cycles, fractal states | AI Agent            | AI Agent             | Expert #2 description and InfraNodus graph setup instructions (Sticky Note2)                                                                                                                                                                |
| The Flow and the Notion Book     | HTTP Request Tool               | InfraNodus expert on creativity, art, dreaming       | AI Agent            | AI Agent             | Expert #3 description, instructions for adding more experts (Sticky Note3)                                                                                                                                                                  |
| The Polysingularity Letters Book | HTTP Request Tool               | InfraNodus expert on networks, polysingularity       | AI Agent            | AI Agent             | Expert #4 description and setup instructions (Sticky Note9)                                                                                                                                                                                |
| OpenAI Model (Disabled)          | LangChain OpenAI Model          | Alternative LLM model (GPT-4o) for precision         | (None)              | (None)               | Comments on model choice and precision (Sticky Note10)                                                                                                                                                                                     |
| Google Gemini Chat Model         | LangChain Google Gemini Model   | Primary LLM model for AI Agent                        | (None)              | AI Agent             | Comments on model speed vs precision (Sticky Note10)                                                                                                                                                                                       |
| Sticky Note                      | Sticky Note                    | Setup instructions for ElevenLabs Conversational AI agent | (None)              | (None)               | Detailed ElevenLabs setup, system prompt, and tool creation instructions (Sticky Note)                                                                                                                                                       |
| Sticky Note7                    | Sticky Note                    | Project overview and tutorial links                   | (None)              | (None)               | Workflow summary, links to tutorial and video (Sticky Note7)                                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: `Webhook`  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `171bf9a6-1390-4195-bd6b-ff3df2e27d1c`)  
   - Set Response Mode to "Respond to Webhook" node  
   - Purpose: Receive user messages from ElevenLabs Conversational AI

2. **Create Simple Memory Node**  
   - Type: `Memory Buffer Window` (LangChain memory)  
   - Configure session key as `={{ $json.body.sessionId }}` (extract from webhook payload)  
   - Set session ID type to `customKey`  
   - Connect input from Webhook, output to AI Agent for context tracking

3. **Create AI Agent Node**  
   - Type: `LangChain Agent`  
   - Input Text: `={{ $json.body.prompt }}` (from webhook payload)  
   - Configure System Message with instructions to:  
     - Consult knowledge experts (InfraNodus graphs)  
     - Always use at least one tool  
     - Adjust queries per tool context  
     - Synthesize responses and highlight discrepancies  
   - Set Prompt Type to `define`  
   - Connect inputs from Webhook and Simple Memory  
   - Connect output to Respond to Webhook node  
   - Credentials: Configure LLM credentials (Google Gemini or OpenAI)

4. **Create Respond to Webhook Node**  
   - Type: `Respond to Webhook`  
   - Default settings  
   - Connect input from AI Agent  
   - Purpose: Send final synthesized response to ElevenLabs agent

5. **Create InfraNodus Expert Nodes (HTTP Request Tool)**  
   For each expert knowledge domain (e.g., Special Agent's Manual, Waves into Patterns, The Flow and the Notion, Polysingularity Letters):  
   - Type: `HTTP Request Tool`  
   - URL: `https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&addStats=true&optimize=develop&includeGraph=false&includeGraphSummary=true&includeStatements=true`  
   - Method: POST  
   - Authentication: HTTP Bearer Auth with InfraNodus API Key (create and assign credentials)  
   - Body Parameters (JSON):  
     - `name`: Expert graph name (e.g., `"waves_into_patterns"`)  
     - `prompt`: Use AI agent override expression `={{ $fromAI('parameters1_Value', 'User\'s request adjusted to suit this context', 'string') }}`  
     - `requestMode`: `"response"`  
     - `aiTopics`: `"true"`  
   - Tool Description: Provide detailed description of expert domain and topics to guide AI agent  
   - Connect output as AI Agent tools inputs

6. **Connect Expert Nodes as Tools to AI Agent**  
   - Assign each HTTP Request node as an `ai_tool` input to the AI Agent node  
   - This allows AI Agent to dynamically route queries to these experts

7. **Configure AI Language Model**  
   - Choose either Google Gemini Chat Model or OpenAI Model (GPT-4o) node  
   - Configure credentials for each (Google Palm API or OpenAI API)  
   - Connect model output to AI Agent's `ai_languageModel` input

8. **Add Sticky Notes for Documentation**  
   - Add sticky notes with:  
     - Instructions for setting up ElevenLabs agent (system prompt, tool naming, webhook linking)  
     - Descriptions of each expert and their knowledge base  
     - Workflow overview and references to tutorials and videos  

9. **Test Integration**  
   - Deploy the workflow and copy the webhook URL  
   - Create ElevenLabs Conversational AI agent:  
     - System prompt referencing the `knowledge_base` tool  
     - Create a tool named `knowledge_base` pointing to the n8n webhook URL  
   - Run test conversations and monitor n8n execution logs for errors

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Full setup instructions for ElevenLabs Conversational AI agent integration, including system prompt example, tool naming (`knowledge_base`), and webhook configuration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | [ElevenLabs AI voice agent setup guide](https://support.noduslabs.com/hc/en-us/articles/20318967066396-How-to-Build-a-Text-Voice-AI-Agent-Chatbot-with-n8n-Elevenlabs-and-InfraNodus) |
| Workflow overview and video tutorial links for building this AI voice agent with InfraNodus experts and ElevenLabs voice chat.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | [Video Tutorial](https://www.youtube.com/watch?v=07-HZZQs5h0)                                                          |
| Expert knowledge domain descriptions are crucial for the AI Agent's tool selection accuracy. Include detailed descriptions and auto-generated Graph RAG summaries from InfraNodus project notes to improve relevance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Inline with Sticky Notes 1, 2, 3, 9                                                                                     |
| Google Gemini Flash Preview models offer faster response times compared to OpenAI GPT models, but OpenAI models may provide better precision in function/expert calling. Choose based on deployment needs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note10                                                                                                          |
| The workflow respects content policies and only uses legal and publicly available data sources.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Disclaimer at start of document                                                                                          |

---

**Disclaimer:** The provided text and workflow originate exclusively from an n8n automated integration and automation setup. It strictly complies with current content policies, contains no illegal or offensive content, and manipulates only legal, public data.