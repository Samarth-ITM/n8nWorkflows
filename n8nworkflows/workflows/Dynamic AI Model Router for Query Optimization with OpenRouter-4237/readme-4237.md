Dynamic AI Model Router for Query Optimization with OpenRouter

https://n8nworkflows.xyz/workflows/dynamic-ai-model-router-for-query-optimization-with-openrouter-4237


# Dynamic AI Model Router for Query Optimization with OpenRouter

---

### 1. Workflow Overview

This workflow, titled **Dynamic AI Model Router for Query Optimization with OpenRouter**, is designed to dynamically select and route user chat queries to the most suitable large language model (LLM) based on the query’s content and intended purpose. It leverages an AI-powered routing agent ("Routing Agent") that analyzes incoming messages and determines which specialized AI model can best handle the request to optimize response quality and performance.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages via a webhook trigger.
- **1.2 AI Model Routing Decision:** Uses a specialized routing agent to analyze the query and decide which LLM to use.
- **1.3 Structured Output Parsing:** Parses the JSON output from the routing agent to extract the selected model and prompt.
- **1.4 Target Model Invocation:** Sends the user's query to the dynamically selected model for final response generation.
- **1.5 Final Response Delivery:** Returns the AI-generated response to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block waits for incoming chat messages from users. It acts as the entry point for the workflow, triggering subsequent processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type:* `@n8n/n8n-nodes-langchain.chatTrigger`  
    - *Role:* Webhook listener node that captures incoming chat messages to initiate the workflow.  
    - *Configuration:* Default options; configured with a unique webhook ID to receive chat input events.  
    - *Input/Output:* No input; outputs the raw chat message data to the next node.  
    - *Edge cases / Failures:* Network connectivity issues, invalid or malformed webhook requests.  
    - *Version:* 1.1  

#### 2.2 AI Model Routing Decision

- **Overview:**  
  This block dynamically determines the best AI model to handle the incoming user query. It uses a custom "Routing Agent," an AI agent node configured with detailed system instructions describing available models and their strengths. The agent outputs a JSON object specifying the prompt and the model to use.

- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - Routing Agent  
  - Sticky Note

- **Node Details:**  
  - **OpenRouter Chat Model**  
    - *Type:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - *Role:* Base language model node that powers the Routing Agent with OpenRouter API, acting as the initial AI chat model.  
    - *Configuration:* Uses OpenRouter API credentials; default options without model override.  
    - *Input/Output:* Receives input from webhook trigger; outputs AI chat response to Structured Output Parser and Routing Agent.  
    - *Edge cases:* API authentication errors, rate limits, connectivity issues.  
    - *Version:* 1  

  - **Routing Agent**  
    - *Type:* `@n8n/n8n-nodes-langchain.agent`  
    - *Role:* Core decision-making AI agent that analyzes the user query and selects the optimal LLM to answer it.  
    - *Configuration:*  
      - System message defining its role as a Routing Agent.  
      - Detailed enumeration of eight AI models it can select from, including their capabilities and ideal use cases.  
      - Output format strictly enforced as a JSON object containing "prompt" and "model" fields.  
    - *Key expression:* System prompt is a large multi-line string defining model selection logic and output format.  
    - *Input/Output:* Receives chat message from "OpenRouter Chat Model"; outputs JSON to "Structured Output Parser" and triggers "AI Agent" with prompt and model selection.  
    - *Edge cases:* Parsing errors if output is not valid JSON, ambiguous model selection, API errors.  
    - *Version:* 1.9  

  - **Sticky Note**  
    - *Type:* `n8n-nodes-base.stickyNote`  
    - *Role:* Documentation embedded within the workflow for human users.  
    - *Content Summary:* Describes the Routing Agent as a dynamic AI-powered routing system that ensures optimized AI responses by selecting the best-suited model based on the query.  
    - *Positioned separately; no connections.*  

#### 2.3 Structured Output Parsing

- **Overview:**  
  This block parses the Routing Agent’s output JSON to extract the selected model name and prompt text for subsequent invocation.

- **Nodes Involved:**  
  - Structured Output Parser

- **Node Details:**  
  - **Structured Output Parser**  
    - *Type:* `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - *Role:* Parses AI output to enforce the expected JSON schema, extracting "prompt" and "model" fields.  
    - *Configuration:*  
      - Schema defined manually as an object with two string properties: "prompt" and "model".  
    - *Input/Output:* Takes AI output JSON string from Routing Agent; outputs parsed JSON for downstream use.  
    - *Edge cases:* Invalid or malformed JSON output from Routing Agent; missing required fields.  
    - *Version:* 1.2  

#### 2.4 Target Model Invocation

- **Overview:**  
  This block sends the user’s original query to the AI model dynamically selected by the Routing Agent, obtaining the final AI-generated answer.

- **Nodes Involved:**  
  - OpenRouter Chat Model1  
  - AI Agent

- **Node Details:**  
  - **OpenRouter Chat Model1**  
    - *Type:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - *Role:* Final AI model invocation node configured to use the model selected dynamically by the Routing Agent.  
    - *Configuration:*  
      - Model name is set dynamically via expression `={{ $json.output.model }}`.  
      - Uses the same OpenRouter API credentials.  
      - No additional options specified.  
    - *Input/Output:* Input is the parsed model selection from Structured Output Parser; outputs to AI Agent node.  
    - *Edge cases:* Invalid model name, API errors, rate limiting.  
    - *Version:* 1  

  - **AI Agent**  
    - *Type:* `@n8n/n8n-nodes-langchain.agent`  
    - *Role:* Executes the prompt against the selected LLM to generate the final AI response.  
    - *Configuration:*  
      - Receives prompt text from parsed output via expression `={{ $json.output.prompt }}`.  
      - Standard agent options, no custom system message.  
    - *Input/Output:* Input is the prompt and model from OpenRouter Chat Model1; output is the final answer (not explicitly shown in this JSON snippet but implied).  
    - *Edge cases:* AI generation failures, timeout, API errors.  
    - *Version:* 1.9  

---

### 3. Summary Table

| Node Name                | Node Type                                    | Functional Role                            | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                           |
|--------------------------|----------------------------------------------|-------------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger        | Entry point; receives chat messages       |                             | Routing Agent              |                                                                                                                                       |
| OpenRouter Chat Model     | @n8n/n8n-nodes-langchain.lmChatOpenRouter   | Base AI model for initial routing agent   | When chat message received  | Routing Agent              |                                                                                                                                       |
| Routing Agent             | @n8n/n8n-nodes-langchain.agent               | AI routing decision maker to select model | OpenRouter Chat Model       | Structured Output Parser, AI Agent | See sticky note: Dynamic Model Selector for Optimal AI Responses - detailed model capabilities and routing instructions          |
| Structured Output Parser  | @n8n/n8n-nodes-langchain.outputParserStructured | Parses routing agent JSON output           | Routing Agent               | OpenRouter Chat Model1     |                                                                                                                                       |
| OpenRouter Chat Model1    | @n8n/n8n-nodes-langchain.lmChatOpenRouter   | Invokes the selected AI model             | Structured Output Parser    | AI Agent                   |                                                                                                                                       |
| AI Agent                 | @n8n/n8n-nodes-langchain.agent               | Generates final AI response using chosen model | OpenRouter Chat Model1    |                            |                                                                                                                                       |
| Sticky Note              | n8n-nodes-base.stickyNote                     | Documentation                             |                             |                            | ## Dynamic Model Selector for Optimal AI Responses: The Agent Decisioner dynamically routes queries to the best-suited model for optimized AI responses. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Chat Trigger** node (`@n8n/n8n-nodes-langchain.chatTrigger`) named "When chat message received."  
   - Configure with default settings; note the webhook URL generated for incoming chat messages.

2. **Add Initial AI Model Node:**  
   - Add an **OpenRouter Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) named "OpenRouter Chat Model."  
   - Assign OpenRouter API credentials (create or reuse an OpenRouter credential with valid API key).  
   - Leave model field empty (default) to allow flexible usage.  
   - Connect output of "When chat message received" node to this node.

3. **Insert Routing Agent Node:**  
   - Add an **Agent** node (`@n8n/n8n-nodes-langchain.agent`) named "Routing Agent."  
   - Set the system message to the detailed multi-line prompt describing:  
     - The role of the Routing Agent.  
     - The list of available AI models with detailed strengths and use cases (eight models total).  
     - The strict JSON output format requirement with fields "prompt" and "model".  
   - Ensure the node is configured to accept input from "OpenRouter Chat Model" via the `ai_languageModel` connection.  
   - Connect "OpenRouter Chat Model" to "Routing Agent" accordingly.

4. **Add Structured Output Parser Node:**  
   - Add an **Output Parser Structured** node (`@n8n/n8n-nodes-langchain.outputParserStructured`) named "Structured Output Parser."  
   - Set schema type to "manual."  
   - Define JSON schema as object with properties:  
     - "prompt": string  
     - "model": string  
   - Connect the `ai_outputParser` output of "Routing Agent" node to this parser node.

5. **Add Final Model Invocation Node:**  
   - Add another **OpenRouter Chat Model** node named "OpenRouter Chat Model1."  
   - Assign the same OpenRouter API credentials.  
   - Set the model field dynamically with expression referencing parsed model: `={{ $json.output.model }}`.  
   - Connect the output of "Structured Output Parser" to this node via `ai_languageModel` connection.

6. **Add Final AI Agent Node:**  
   - Add an **Agent** node named "AI Agent."  
   - Configure the text input with expression: `={{ $json.output.prompt }}` to use the prompt extracted from the parser.  
   - Connect the output of "OpenRouter Chat Model1" node to this node via `ai_languageModel` connection.

7. **Add Sticky Note for Documentation:**  
   - Add a **Sticky Note** node with content describing the workflow’s purpose as a dynamic model selector to optimize AI responses by routing queries to specialized models.  
   - Place it visually near the Routing Agent node for clarity.

8. **Verify Connections:**  
   - Ensure connections flow as:  
     - When chat message received → OpenRouter Chat Model → Routing Agent → Structured Output Parser → OpenRouter Chat Model1 → AI Agent.

9. **Credential Setup:**  
   - Set up OpenRouter API credentials with valid API keys for all OpenRouter Chat Model nodes.

10. **Testing:**  
    - Test the webhook URL by sending sample chat messages.  
    - Confirm that the Routing Agent returns valid JSON with "prompt" and "model."  
    - Confirm the final AI Agent produces appropriate answers using the dynamically selected model.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses the OpenRouter API for accessing multiple LLMs and dynamically routing queries to optimize response relevance and efficiency.                               | OpenRouter API documentation: https://openrouter.ai/docs                                         |
| The Routing Agent system message carefully defines multiple popular AI models with distinct strengths to ensure appropriate model selection based on query type and complexity. | Embedded within the "Routing Agent" node system message.                                         |
| For enhanced debugging, watch for JSON parsing errors in the Structured Output Parser node caused by malformed routing outputs.                                               | Structured Output Parser configuration enforces JSON schema for robustness.                      |
| The workflow is designed with modularity in mind, enabling easy extension by adding new models or modifying routing logic in the Routing Agent’s prompt.                       | Can be enhanced with additional nodes or external APIs if needed.                                |
| For secure operation, ensure OpenRouter API credentials are stored securely within n8n credentials manager.                                                                     | n8n Credentials setup guidelines: https://docs.n8n.io/credentials/                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.

---