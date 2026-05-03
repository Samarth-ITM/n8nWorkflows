Enhance AI Chatbot Responses with InfraNodus Knowledge Graph Reasoning & GraphRAG

https://n8nworkflows.xyz/workflows/enhance-ai-chatbot-responses-with-infranodus-knowledge-graph-reasoning---graphrag-6887


# Enhance AI Chatbot Responses with InfraNodus Knowledge Graph Reasoning & GraphRAG

---

### 1. Workflow Overview

This workflow is designed to enhance AI chatbot responses by integrating reasoning capabilities from InfraNodus knowledge graphs combined with GraphRAG technology. It targets use cases where AI chatbot answers are augmented by domain-specific ontologies and knowledge graph reasoning, improving answer relevance and depth beyond simple language model completions.

The workflow logically divides into three main blocks:

- **1.1 Input Reception:** Triggers on receiving a chat message, initiating the processing chain.
- **1.2 Reasoning Ontology Augmentation:** Sends the user’s original query to InfraNodus with a reasoning ontology to reformulate the prompt using graph-based logic.
- **1.3 Knowledge Base Query with GraphRAG:** Uses the augmented query to query InfraNodus’ knowledge base via GraphRAG for a reasoned, context-aware response.

Two sticky notes provide conceptual guidance and references for each block, helping users understand the reasoning approach and knowledge graph setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a chat message is received, capturing the user input that will be processed and augmented downstream.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type & Role:* LangChain Chat Trigger node; entry point for chat input.  
    - *Configuration:* Public webhook enabled, allowing external systems or web clients to send chat messages that trigger this workflow. No additional options configured.  
    - *Expressions/Variables:* Captures incoming chat input as `$json.chatInput`.  
    - *Connections:* Output connected to "Prompt Augmented with Reasoning Ontology" node.  
    - *Version:* 1.1  
    - *Potential Failures:* Webhook not reachable, invalid or empty chat message payloads, malformed requests.  
    - *Sub-workflow:* None.

#### 2.2 Reasoning Ontology Augmentation

- **Overview:**  
  Reformulates the user’s original query by sending it to InfraNodus API with a reasoning ontology. This node leverages the knowledge graph’s reasoning logic to generate an enhanced prompt representing the user’s intent in a richer semantic context.

- **Nodes Involved:**  
  - Prompt Augmented with Reasoning Ontology

- **Node Details:**

  - **Prompt Augmented with Reasoning Ontology**  
    - *Type & Role:* HTTP Request node; sends POST request to InfraNodus API endpoint for graph-based prompt reformulation.  
    - *Configuration:*  
      - URL: InfraNodus API endpoint `/graphAndAdvice` with parameters to avoid saving, include stats, enable optimization and statements.  
      - Method: POST  
      - Body: JSON parameters include:  
        - `name`: `"eightos_system"` (name of the InfraNodus knowledge graph to use, referencing a reasoning ontology).  
        - `requestMode`: `"reprompt"` instructing the API to reformulate the prompt.  
        - `aiTopics`: `"true"` enables topic extraction.  
        - `prompt`: Expression `={{ $json.chatInput }}` to send the original user input.  
        - `systemPrompt`: Instructional text telling the API to reformulate the query using provided context.  
      - Authentication: HTTP Bearer token credential named "InfraNodus Expert".  
    - *Expressions/Variables:* Uses `$json.chatInput` from trigger node.  
    - *Connections:* Output connected to "Ask the Knowledge Base" node.  
    - *Version:* 4.2  
    - *Potential Failures:* HTTP errors (timeout, 4xx/5xx), invalid token/auth errors, API rate limits, malformed responses, missing or incorrect graph name, empty or invalid user prompt.  
    - *Sub-workflow:* None.  
    - *Notes:* This node implements the core reasoning augmentation step by interacting with InfraNodus’ reasoning ontology graph.

#### 2.3 Knowledge Base Query with GraphRAG

- **Overview:**  
  Takes the augmented prompt from the previous step and queries the InfraNodus knowledge base using GraphRAG, which traverses the knowledge graph to extract a logically reasoned response for the user query.

- **Nodes Involved:**  
  - Ask the Knowledge Base

- **Node Details:**

  - **Ask the Knowledge Base**  
    - *Type & Role:* HTTP Request node; sends POST request to InfraNodus API endpoint to retrieve a response based on the augmented prompt and graph traversal via GraphRAG.  
    - *Configuration:*  
      - URL: Same InfraNodus API endpoint `/graphAndAdvice` with similar query parameters as the previous node.  
      - Method: POST  
      - Body: JSON parameters include:  
        - `name`: `"eightos_system"` (same or potentially different knowledge graph name to query).  
        - `requestMode`: `"response"` instructing the API to generate an answer.  
        - `aiTopics`: `"true"` to use topics extraction.  
        - `prompt`: Expression `={{ $json.aiAdvice[0].text }}` that uses the reformulated query text from prior node’s output (`aiAdvice[0].text`).  
        - `systemPrompt`: Instruction clarifying that the context is to be used as reasoning logic, not direct content for the answer.  
      - Authentication: HTTP Bearer token credential named "InfraNodus Expert".  
    - *Expressions/Variables:* Uses previous node output `aiAdvice[0].text`.  
    - *Connections:* Terminal node (no outputs connected).  
    - *Version:* 4.2  
    - *Potential Failures:* HTTP errors, auth errors, invalid or unexpected response structures, empty input from prior node, API limits.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                          | Node Type                           | Functional Role                            | Input Node(s)                  | Output Node(s)                        | Sticky Note                                                                                                                   |
|----------------------------------|-----------------------------------|--------------------------------------------|-------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When chat message received        | @n8n/n8n-nodes-langchain.chatTrigger | Entry trigger capturing user chat message | (Trigger)                     | Prompt Augmented with Reasoning Ontology | ## 1. Trigger the chat and send a message You can also make this node publicly available via a URL and embed it on a website... |
| Prompt Augmented with Reasoning Ontology | n8n-nodes-base.httpRequest          | Reformulates user query using reasoning ontology graph | When chat message received     | Ask the Knowledge Base               | ## 2. Reasoning Expert Reformulates the User's Query: Use InfraNodus graph with reasoning ontology for prompt augmentation...   |
| Ask the Knowledge Base           | n8n-nodes-base.httpRequest          | Queries InfraNodus knowledge base with augmented prompt via GraphRAG | Prompt Augmented with Reasoning Ontology | (None)                             | ## 3. The augmented query is sent to the knowledge base and the response is retrieved using GraphRAG...                        |
| Sticky Note                     | n8n-nodes-base.stickyNote           | Conceptual explanation and workflow overview | (None)                        | (None)                              | ## AI Chatbot Agent with Experts Uses InfraNodus knowledge graphs and GraphRAG for high-quality AI chatbot answers...          |
| Sticky Note1                    | n8n-nodes-base.stickyNote           | Explanation of reasoning ontology augmentation step | (None)                        | (None)                              | ## 2. Reasoning Expert Reformulates the User's Query: Guidance and references on creating reasoning ontologies...               |
| Sticky Note9                    | n8n-nodes-base.stickyNote           | Explanation of knowledge base query step | (None)                        | (None)                              | ## 3. The augmented query is sent to the knowledge base and the response is retrieved using GraphRAG guidance and references... |
| Sticky Note2                    | n8n-nodes-base.stickyNote           | Explanation of chat trigger usage          | (None)                        | (None)                              | ## 1. Trigger the chat and send a message: Info about public webhook and Telegram integration possibilities                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add a **LangChain Chat Trigger** node named `When chat message received`.  
   - Configure it with:  
     - Public webhook enabled to allow external chat message reception.  
     - No additional options needed.  
   - This node receives the raw user chat input as `$json.chatInput`.

2. **Create the Reasoning Ontology Augmentation Node**  
   - Add an **HTTP Request** node named `Prompt Augmented with Reasoning Ontology`.  
   - Set method to `POST`.  
   - Set URL to:  
     ```
     https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&addStats=true&optimize=develop&includeStatements=true&includeGraphSummary=true&includeGraph=false
     ```  
   - In the body parameters (JSON):  
     - `name`: `"eightos_system"` (replace with your InfraNodus graph name containing the reasoning ontology)  
     - `requestMode`: `"reprompt"`  
     - `aiTopics`: `"true"`  
     - `prompt`: Use expression to map from trigger node: `={{ $json.chatInput }}`  
     - `systemPrompt`: `"Your task is to reformulate the original query of a user using the context provided"`  
   - Authentication: Use HTTP Bearer Authentication credentials with your InfraNodus API key (named e.g. "InfraNodus Expert").  
   - Connect the trigger node output to this node input.

3. **Create the Knowledge Base Query Node**  
   - Add an **HTTP Request** node named `Ask the Knowledge Base`.  
   - Set method to `POST`.  
   - Use the same base URL as previous node with identical query parameters.  
   - In the body parameters (JSON):  
     - `name`: `"eightos_system"` (or another knowledge graph name if querying a different graph)  
     - `requestMode`: `"response"`  
     - `aiTopics`: `"true"`  
     - `prompt`: Use expression to extract from previous node output: `={{ $json.aiAdvice[0].text }}`  
     - `systemPrompt`: `"Use the context you are provided as a logic to use when providing a response to the user query, not as the content you should be providing. IT IS IMPERATIVE THAT YOU DO NOT EXTRACT THE CONTENT FROM THE CONTEXT PROVIDED FOR YOUR ANSWER BUT USE IT AS A REASONING LOGIC TO PROVIDE YOUR ANSWER."`  
   - Authentication: Use the same InfraNodus HTTP Bearer credentials.  
   - Connect the output of the reasoning augmentation node to this node input.

4. **(Optional) Add Sticky Note Nodes**  
   - Add sticky notes to document and explain each step for easier maintenance. Use the content from the provided sticky notes if desired.

5. **Credentials Setup**  
   - Create an HTTP Bearer Authentication credential in n8n with your InfraNodus API key.  
   - Assign this credential to both HTTP Request nodes.

6. **Testing**  
   - Activate the workflow.  
   - Send a test chat message to the webhook URL of the trigger node.  
   - Inspect subsequent nodes to verify prompt reformulation and final response retrieval.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Use your [InfraNodus graph](https://infranodus.com) as the knowledge base for your AI chatbot. Upload data, generate knowledge graphs, and connect them as experts for reasoning-enhanced chatbot responses.                     | InfraNodus main site: https://infranodus.com                                                             |
| Video demo showcasing the workflow and InfraNodus integration: [https://www.youtube.com/watch?v=kS0QTUvcH6E](https://www.youtube.com/watch?v=kS0QTUvcH6E)                                                                     | YouTube video                                                                                            |
| Detailed documentation on using InfraNodus knowledge graphs as experts for AI chatbots: [Nodus Labs support portal](https://support.noduslabs.com/hc/en-us/articles/20174217658396-Using-InfraNodus-Knowledge-Graphs-as-Experts-for-AI-Chatbot-Agents-in-n8n) | InfraNodus support portal article                                                                        |
| Guidance on creating reasoning ontologies and knowledge graphs with InfraNodus: [Article on reasoning agents](https://support.noduslabs.com/hc/en-us/articles/21429518472988-Using-Knowledge-Graphs-as-Reasoning-Experts)        | InfraNodus support portal article                                                                        |
| Use the [InfraNodus AI Ontologies Generator](https://infranodus.com/import/ai-ontologies) to create reasoning chain graphs.                                                                                                  | InfraNodus AI Ontologies Generator                                                                       |
| Repository of free knowledge graphs for various domains on InfraNodus: [https://infranodus.com/knowledge-graphs](https://infranodus.com/knowledge-graphs)                                                                     | InfraNodus knowledge graphs repository                                                                   |
| Example graph (EightOS cognitive variability framework): [https://infranodus.com/expert/eightos_system](https://infranodus.com/expert/eightos_system?background=dark&show_analytics=1&most_influential=bc2&maxnodes=150&threshold=8&labelsize=proportional&edgestype=curve&drawedges=true&drawnodes=true&labelsizeratio=2&dynamic=highlight&cutgraph=1&selected=highlight) | Example InfraNodus reasoning graph                                                                        |
| Telegram integration example for triggering chat messages: [n8n workflow 4485](https://n8n.io/workflows/4485-telegram-ai-chatbot-agent-with-infranodus-graphrag-knowledge-base/)                                                | n8n workflow example for Telegram chatbot trigger                                                       |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a powerful integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.

---