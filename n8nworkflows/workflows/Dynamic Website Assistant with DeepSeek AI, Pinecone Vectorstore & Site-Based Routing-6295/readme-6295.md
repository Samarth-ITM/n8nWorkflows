Dynamic Website Assistant with DeepSeek AI, Pinecone Vectorstore & Site-Based Routing

https://n8nworkflows.xyz/workflows/dynamic-website-assistant-with-deepseek-ai--pinecone-vectorstore---site-based-routing-6295


# Dynamic Website Assistant with DeepSeek AI, Pinecone Vectorstore & Site-Based Routing

### 1. Workflow Overview

This workflow implements a **Dynamic Website Assistant** that provides contextual AI-driven responses tailored to multiple different websites (or "sites") and their specific pages, using DeepSeek AI language models, Pinecone vector stores for contextual retrieval, and Postgres for chat memory. It routes user queries dynamically based on the `site` parameter in the incoming webhook request, selecting the appropriate AI agent, vector stores, and chat memory tailored for that site.

The workflow’s core logic is organized into these logical blocks:

- **1.1 Input Reception**: Receives user queries via a webhook, capturing user ID, site, page, and query text.
- **1.2 Site-Based Routing**: Routes queries to specific AI agents based on the `site` parameter.
- **1.3 AI Agents and Contextual Processing**: Each agent uses a custom system prompt and leverages multiple Pinecone vector stores (each embedding vectors from different knowledge domains like services, policies, packages) plus Postgres chat memory to produce context-aware answers.
- **1.4 Response Dispatch**: Sends the AI-generated answers back to the user via webhook response nodes.

Each AI Agent block is a self-contained processing pipeline involving a Cohere embeddings node, multiple Pinecone vector stores as tools, a Postgres chat memory node, and an OpenRouter language model node. The agents are distinguished primarily by the site they handle and the specific vector namespaces they query.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Receives incoming user queries as HTTP requests, extracting parameters like `userId`, `site`, `page`, and `query`.
  
- **Nodes Involved:**  
  - Webhook  
  - Sticky Note (comment: "accepts user input")

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook (Trigger)  
    - Configuration: Listens on path `f22b5848-6722-45fb-ba0c-b874f5813b5c`  
    - Inputs: HTTP Request with query parameters including `userId`, `site`, `page`, `query`  
    - Outputs: JSON object with these parameters for downstream processing  
    - Edge Cases: Missing parameters, malformed requests, unsupported HTTP methods  
    - Sticky Note: "accepts user input"

#### 2.2 Site-Based Routing

- **Overview:**  
  Routes the workflow execution to one of three AI agent pipelines depending on the `site` parameter value.  
  Supports sites: `"test_site"`, `"test_site2"`, and `"test_site3"`.  

- **Nodes Involved:**  
  - Switch  
  - Sticky Note (comment: "switch to specific agent according to site")

- **Node Details:**

  - **Switch**  
    - Type: Switch (Conditional Branching)  
    - Configuration: Three rules based on `$json.query.site` equals `"test_site"`, `"test_site2"`, or `"test_site3"`  
    - Inputs: Output from Webhook node  
    - Outputs: Branch 1 → `"test_site"` → AI Agent1 ghostwritingpartner  
                Branch 2 → `"test_site2"` → AI Agent ebook-wr  
                Branch 3 → `"test_site3"` → AI Agent for groton  
    - Edge Cases: Unknown or missing `site` values lead to no output branch (workflow halts or errors)  
    - Sticky Note: "switch to specific agent according to site"

#### 2.3 AI Agent Pipelines and Contextual Processing

Each AI Agent block contains the following subcomponents:

- Postgres Chat Memory node for session-contextual chat history storage and retrieval.
- OpenRouter Chat Model node using DeepSeek AI chat model.
- Multiple Cohere Embeddings nodes paired with Pinecone Vector Store nodes, each corresponding to a specific knowledge domain vectorstore namespace.
- AI Agent node that integrates the above as memory, language model, and retrieval tools, with a custom system prompt dynamically referencing the `site` and `page` parameters.
- Respond to Webhook node to output the final answer.

##### 2.3.1 AI Agent1 ghostwritingpartner (for site = "test_site")

- **Overview:**  
  Handles queries for `test_site`, using multiple vector stores covering services, packages, policies, and contact info. Uses a custom system prompt tailored for this site.
  
- **Nodes Involved:**  
  - Postgres Chat Memory1  
  - OpenRouter Chat Model1  
  - Embeddings Cohere8  
  - Pinecone Vector Store8  
  - Respond to Webhook1  
  - AI Agent1 ghostwritingpartner  
  - Sticky Note4 (comment: "same goes for remaining agents as above")

- **Node Details:**

  - **Postgres Chat Memory1**  
    - Role: Maintains chat history per user (session key from `userId`) in Postgres table `n8n_chat_histories_test_1`  
    - Inputs: Webhook data  
    - Outputs: Provides memory to AI Agent  
    - Credentials: Postgres account  
    - Potential Failures: DB connection issues, invalid session keys

  - **OpenRouter Chat Model1**  
    - Role: LLM backend using DeepSeek chat model  
    - Model: `deepseek/deepseek-chat-v3-0324:free`  
    - Inputs: Prompt from AI Agent  
    - Credentials: OpenRouter API  
    - Failures: API auth errors, rate limits

  - **Embeddings Cohere8**  
    - Role: Generates embedding vectors for query text  
    - Credentials: Cohere API  
    - Outputs: Sent to Pinecone Vector Store8

  - **Pinecone Vector Store8**  
    - Role: Retrieves top 6 relevant documents from namespace `"ghost_wr"` using embeddings  
    - Index: Specific Pinecone index ID  
    - Credentials: Pinecone API  
    - Used as retrieval tool in AI Agent

  - **AI Agent1 ghostwritingpartner**  
    - Role: Core AI logic node combining query text, system prompt, vector store tools, memory, and LLM  
    - System Prompt: Customized with `site` and `page` values, instructs AI to answer only from vector store content, following strict guidelines and formatting rules  
    - Inputs: Query text from webhook, memory, vector store tools, language model  
    - Outputs: Generated answer JSON for response node

  - **Respond to Webhook1**  
    - Role: Returns JSON response `{"answer": <AI output>}` back to the user  
    - Inputs: AI Agent1 output  
    - Failures: Network errors, client disconnects

##### 2.3.2 AI Agent ebook-wr (for site = "test_site2")

- **Overview:**  
  Similar to agent above but for `test_site2`, accessing vector stores under namespaces related to ebook writing and publishing.  
  Uses dedicated Postgres table and credentials.

- **Nodes Involved:**  
  - Postgres Chat Memory2  
  - OpenRouter Chat Model2  
  - Embeddings Cohere11  
  - Pinecone Vector Store11  
  - AI Agent ebook-wr  
  - Respond to Webhook2

- **Node Details:**  
  Similar roles and configurations as above, with different Postgres table `n8n_chat_histories_test_2` and Pinecone namespaces like `"ebook_wr"`.

##### 2.3.3 AI Agent for groton (for site = "test_site3")

- **Overview:**  
  Handles queries for `test_site3`, uses a set of vector stores for services, packages, policies, privacy, etc.  
  Employs separate Postgres chat memory table and OpenRouter credentials.

- **Nodes Involved:**  
  - Postgres Chat Memory  
  - OpenRouter Chat Model  
  - Embeddings Cohere (several nodes, e.g. Embeddings Cohere, Cohere6, Cohere7, Cohere9, Cohere10)  
  - Pinecone Vector Store5, 6, 7, 9, 10 (multiple namespaces for different knowledge domains)  
  - AI Agent for groton  
  - Respond to Webhook

- **Node Details:**  
  - Complex system prompt tailored with detailed instructions to respond only with vector store data, respecting the page context  
  - Multiple vector stores used as tools for retrieval from distinct namespaces such as `"Service_Packages_Overview"`, `"services"`, `"terms_of_use"`, `"Privacy_Policy"`, `"refund_policy"`  
  - Postgres table `n8n_chat_histories_test_3` for chat memory  
  - OpenRouter model `deepseek-chat-v3-0324:free` with API credentials  
  - Failures: API limits, DB connection issues, embedding service errors

#### 2.4 Response Dispatch

- **Overview:**  
  Each AI agent’s answer is sent back as a JSON response to the original webhook caller.

- **Nodes Involved:**  
  - Respond to Webhook  
  - Respond to Webhook1  
  - Respond to Webhook2  
  - Sticky Note3 ("sends response back to chat")

- **Node Details:**

  - These nodes take the output from the respective AI agent and respond with JSON: `{"answer": <AI output>}`  
  - Type: Respond to Webhook (HTTP Response)  
  - Edge Cases: Client disconnect before response, malformed AI output

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                             | Input Node(s)          | Output Node(s)           | Sticky Note                                      |
|-------------------------|------------------------------------|--------------------------------------------|------------------------|--------------------------|-------------------------------------------------|
| Webhook                 | HTTP Webhook (Trigger)              | Receives user query input                   | -                      | Switch                   | accepts user input                               |
| Switch                  | Switch                             | Routes query based on `site` parameter     | Webhook                | AI Agent1 ghostwritingpartner, AI Agent ebook-wr, AI Agent for groton | switch to specific agent according to site      |
| AI Agent1 ghostwritingpartner | Langchain Agent                  | Processes queries for `test_site`           | Switch, Postgres Chat Memory1, OpenRouter Chat Model1, Pinecone Vector Store8 | Respond to Webhook1          | same goes for remaining agents as above         |
| Postgres Chat Memory1   | Postgres Chat Memory                | Stores/retrieves chat history for `test_site` | -                      | AI Agent1 ghostwritingpartner |                                                 |
| OpenRouter Chat Model1  | Language Model (OpenRouter)         | LLM for `test_site`                         | -                      | AI Agent1 ghostwritingpartner |                                                 |
| Embeddings Cohere8      | Embeddings (Cohere API)             | Generates embeddings for vector retrieval | -                      | Pinecone Vector Store8    |                                                 |
| Pinecone Vector Store8  | Vector Store (Pinecone)             | Retrieves context vectors for `ghost_wr`  | Embeddings Cohere8      | AI Agent1 ghostwritingpartner |                                                 |
| Respond to Webhook1     | Respond to Webhook                  | Sends AI response to user                   | AI Agent1 ghostwritingpartner | -                    | sends response back to chat                      |
| AI Agent ebook-wr       | Langchain Agent                    | Processes queries for `test_site2`          | Switch, Postgres Chat Memory2, OpenRouter Chat Model2, Pinecone Vector Store11 | Respond to Webhook2          | same goes for remaining agents as above         |
| Postgres Chat Memory2   | Postgres Chat Memory                | Stores/retrieves chat history for `test_site2` | -                      | AI Agent ebook-wr          |                                                 |
| OpenRouter Chat Model2  | Language Model (OpenRouter)         | LLM for `test_site2`                        | -                      | AI Agent ebook-wr          |                                                 |
| Embeddings Cohere11     | Embeddings (Cohere API)             | Generates embeddings for vector retrieval | -                      | Pinecone Vector Store11   |                                                 |
| Pinecone Vector Store11 | Vector Store (Pinecone)             | Retrieves context vectors for `ebook_wr`  | Embeddings Cohere11     | AI Agent ebook-wr          |                                                 |
| Respond to Webhook2     | Respond to Webhook                  | Sends AI response to user                   | AI Agent ebook-wr         | -                        | sends response back to chat                      |
| AI Agent for groton     | Langchain Agent                    | Processes queries for `test_site3`          | Switch, Postgres Chat Memory, OpenRouter Chat Model, multiple Pinecone Vector Stores | Respond to Webhook          | same goes for remaining agents as above         |
| Postgres Chat Memory    | Postgres Chat Memory                | Stores/retrieves chat history for `test_site3` | -                      | AI Agent for groton        |                                                 |
| OpenRouter Chat Model   | Language Model (OpenRouter)         | LLM for `test_site3`                        | -                      | AI Agent for groton        |                                                 |
| Embeddings Cohere       | Embeddings (Cohere API)             | Generates embeddings for vector retrieval | -                      | Pinecone Vector Store5,6,7,9,10 |                                                 |
| Pinecone Vector Store5  | Vector Store (Pinecone)             | Retrieves context vectors for `Service_Packages_Overview` | Embeddings Cohere       | AI Agent for groton        |                                                 |
| Pinecone Vector Store6  | Vector Store (Pinecone)             | Retrieves context vectors for `services`  | Embeddings Cohere6      | AI Agent for groton        |                                                 |
| Pinecone Vector Store7  | Vector Store (Pinecone)             | Retrieves context vectors for `terms_of_use` | Embeddings Cohere7      | AI Agent for groton        |                                                 |
| Pinecone Vector Store9  | Vector Store (Pinecone)             | Retrieves context vectors for `Privacy_Policy` | Embeddings Cohere9      | AI Agent for groton        |                                                 |
| Pinecone Vector Store10 | Vector Store (Pinecone)             | Retrieves context vectors for `refund_policy` | Embeddings Cohere10     | AI Agent for groton        |                                                 |
| Respond to Webhook      | Respond to Webhook                  | Sends AI response to user                   | AI Agent for groton       | -                        | sends response back to chat                      |
| Sticky Note             | Sticky Note                        | Visual comment: "accepts user input"       | -                      | -                        |                                                 |
| Sticky Note1            | Sticky Note                        | Visual comment: "switch to specific agent according to site" | -                      | -                        |                                                 |
| Sticky Note2            | Sticky Note                        | Visual comment describing AI Agent architecture | -                      | -                        | 🧠 AI Agent: Dynamically tailors responses per website/page |
| Sticky Note3            | Sticky Note                        | Visual comment: "sends response back to chat" | -                      | -                        |                                                 |
| Sticky Note4            | Sticky Note                        | Visual comment: "same goes for remaining agents as above" | -                      | -                        |                                                 |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow in n8n manually, follow these steps:

**1. Create Webhook Node**  
- Type: Webhook  
- Path: `f22b5848-6722-45fb-ba0c-b874f5813b5c`  
- HTTP Method: POST (default)  
- Purpose: Receives JSON with `userId`, `site`, `page`, `query`

**2. Create Switch Node**  
- Type: Switch  
- Input: Connect from Webhook output  
- Add 3 rules with condition type "String equals":  
  - If `{{$json.query.site}} == "test_site"` → Output 1  
  - If `{{$json.query.site}} == "test_site2"` → Output 2  
  - If `{{$json.query.site}} == "test_site3"` → Output 3

**3. For each site, create an AI Agent pipeline:**

---

#### AI Agent Pipeline Template (repeat for each site with specific parameters)

**3.a. Postgres Chat Memory Node**  
- Type: Postgres Chat Memory (Langchain)  
- Table Name: e.g., `n8n_chat_histories_test_1` for `test_site`  
- Session Key: Expression `{{$node["Webhook"].json["query"]["userId"]}}`  
- Credentials: Select Postgres credentials

**3.b. OpenRouter Chat Model Node**  
- Type: OpenRouter Chat Model (Langchain)  
- Model: `deepseek/deepseek-chat-v3-0324:free`  
- Credentials: OpenRouter API credential for the site

**3.c. Cohere Embeddings Node(s)**  
- Type: Embeddings Cohere (Langchain)  
- Credentials: Cohere API  
- No special parameters; default config

**3.d. Pinecone Vector Store Node(s)**  
- Type: Vector Store Pinecone (Langchain)  
- Mode: `retrieve-as-tool`  
- TopK: 6 (if applicable)  
- Pinecone Namespace: site-specific (e.g., `ghost_wr` for `test_site`)  
- Pinecone Index: Provide index ID for the namespace  
- Credentials: Pinecone API  
- Tool Description: Describe the store’s content (e.g., "vector store for: Service Packages Overview")

**3.e. AI Agent Node**  
- Type: Langchain Agent  
- Text Input: Expression `{{$node["Webhook"].json["query"]["query"]}}`  
- System Prompt: Use a custom prompt referencing `{{$json.query.site}}` and `{{$json.query.page}}` dynamically. Include instructions to answer only from vector store content, format responses accordingly, and end replies with `— max`.  
- Attach as AI Memory: Postgres Chat Memory node  
- Attach as AI Language Model: OpenRouter Chat Model node  
- Attach as AI Tool(s): Connect all relevant Pinecone Vector Stores  
- Credentials: None needed here (uses linked nodes’ credentials)

**3.f. Respond to Webhook Node**  
- Type: Respond to Webhook  
- Respond With: JSON  
- Response Body: Expression `{ "answer": {{$json["output"]}} }`  
- Input: Connect from AI Agent node output

---

**4. Connect Nodes**

- Connect Webhook → Switch  
- Switch output to AI Agent pipeline input (Postgres Chat Memory, OpenRouter Chat Model, Embeddings Cohere(s))  
- Connect Embeddings Cohere output → corresponding Pinecone Vector Store(s) input  
- Connect Postgres Chat Memory → AI Agent (memory input)  
- Connect OpenRouter Chat Model → AI Agent (language model input)  
- Connect Pinecone Vector Stores → AI Agent (tool input)  
- Connect AI Agent → Respond to Webhook

---

**5. Credential Setup**

- Postgres: Provide connection details (host, port, user, password, database) to all Postgres Chat Memory nodes.  
- OpenRouter API: Provide API keys for each OpenRouter Chat Model node.  
- Cohere API: Provide API keys for all Embeddings Cohere nodes.  
- Pinecone API: Provide API keys and environment details for all Pinecone Vector Store nodes.

---

**6. System Prompt Customization**

- Customize each AI Agent’s system prompt with the relevant site and page placeholders.  
- Include detailed instructions about the knowledge domains, vector stores to query, response formatting rules, and forbidden questions.

---

**7. Testing**

- Deploy workflow.  
- Send HTTP POST requests to webhook URL with JSON body including keys: `userId`, `site`, `page`, `query`.  
- Verify correct routing, AI agent response, and response format.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The AI agents use DeepSeek’s `deepseek-chat-v3-0324:free` model via OpenRouter API for advanced contextual chat capabilities.     | OpenRouter API documentation recommended for troubleshooting API keys and rate limits.              |
| Cohere embeddings are used to vectorize incoming queries before querying Pinecone vector stores; ensure Cohere API keys have quota.| Cohere API docs: https://docs.cohere.ai/                                                             |
| Pinecone vector stores are separated by namespace per knowledge domain and site; maintain index IDs carefully for proper routing.| Pinecone documentation: https://www.pinecone.io/docs/                                               |
| Postgres chat memory tables are separate per site to isolate user session histories.                                               | Database schema must support the chat history table structure expected by n8n Langchain nodes.     |
| Site-based routing allows adding new sites by replicating the AI agent pipeline with appropriate namespaces and credentials.      | The Switch node can be updated with new site rules accordingly.                                     |
| System prompts enforce strict rules to avoid hallucinations by restricting responses only to vector store content.                | Prompt engineering best practices apply here; consider prompt length limits of the model.           |
| Sticky notes in the workflow provide contextual explanations useful for maintenance and onboarding.                                | n8n sticky notes are used for visual documentation inside the workflow editor.                      |

---

**Disclaimer:** The text provided is exclusively extracted from an automated n8n workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.