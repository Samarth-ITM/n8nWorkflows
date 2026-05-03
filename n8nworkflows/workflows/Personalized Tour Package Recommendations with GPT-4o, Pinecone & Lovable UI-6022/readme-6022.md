Personalized Tour Package Recommendations with GPT-4o, Pinecone & Lovable UI

https://n8nworkflows.xyz/workflows/personalized-tour-package-recommendations-with-gpt-4o--pinecone---lovable-ui-6022


# Personalized Tour Package Recommendations with GPT-4o, Pinecone & Lovable UI

### 1. Workflow Overview

This workflow delivers **Personalized Tour Package Recommendations** by integrating advanced AI models (OpenAI GPT-4o), a Pinecone vector database for tour package embeddings, and a user-facing web UI via Lovable form integration. It is designed for travel agencies or tour operators aiming to provide customized travel itineraries based on users’ destination inputs or activity preferences.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user requests via a webhook connected to Lovable UI.
- **1.2 AI Agent Processing:** Uses a Langchain AI Agent with GPT-4o to interpret user inputs and orchestrate the recommendation process.
- **1.3 Vector Store Search:** Queries the Pinecone vector database containing pre-embedded tour packages to find relevant matches.
- **1.4 AI Response Composition:** Generates a user-friendly, polite, and emoji-enhanced reply with customized tour packages.
- **1.5 Structured Output Formatting:** Converts AI-generated content into a strict JSON itinerary format for frontend consumption.
- **1.6 Response Delivery:** Sends the structured response back to the Lovable UI via the webhook response node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block receives HTTP POST requests from the Lovable UI web form, triggering the workflow with user input such as destination or activity search terms.
- **Nodes Involved:** `Webhook`, `Respond to Webhook`
- **Node Details:**

  - **Webhook**
    - Type: HTTP Webhook Listener
    - Role: Entry point for external requests; listens for POST requests at a specific path.
    - Configuration:  
      - HTTP Method: POST  
      - Path: `12b44ee5-c43e-430c-a1d4-4fc5ff5e45c4` (unique webhook endpoint)  
      - Response Mode: Uses downstream node for response
    - Inputs: External HTTP requests
    - Outputs: Passes request data to `Tour Recommendation AI Agent`
    - Edge Cases: Incorrect method calls, invalid JSON payloads, missing required fields in input

  - **Respond to Webhook**
    - Type: HTTP Response Node
    - Role: Sends JSON response back to the Lovable UI client
    - Configuration:  
      - Respond with JSON  
      - Response Body: Outputs the `output` property from the AI Agent’s JSON response
    - Inputs: Receives processed recommendation data from `Tour Recommendation AI Agent`
    - Outputs: HTTP response to caller
    - Edge Cases: Empty or malformed output JSON, timeout if AI Agent fails

---

#### 2.2 AI Agent Processing

- **Overview:** Orchestrates the core AI logic by interpreting user queries, interacting with the vector store, and managing the response format.
- **Nodes Involved:** `Tour Recommendation AI Agent`, `Structured Output Parser`
- **Node Details:**

  - **Tour Recommendation AI Agent**
    - Type: Langchain AI Agent Node
    - Role: Central AI orchestrator that leverages GPT-4o and external tools (vector store) to generate tour recommendations.
    - Configuration:
      - Input text: Extracted from webhook JSON body property `destination`  
      - System message: Defines the role as an experienced travel agent using Pinecone for tour data  
      - Output parser: Enabled with Structured Output Parser integration  
      - Prompt Type: Define (custom prompt with system instructions)
    - Inputs: User query from `Webhook` node
    - Outputs: Structured tour package recommendation JSON forwarded to `Respond to Webhook`
    - Version: 1.9
    - Edge Cases:  
      - Missing or invalid destination input  
      - API rate limits or timeouts  
      - Parsing errors if AI output deviates from expected structure  
      - Pinecone connection failures (via tool integration)

  - **Structured Output Parser**
    - Type: Langchain Structured Output Parser
    - Role: Ensures AI responses conform to a strict JSON schema for itinerary details.
    - Configuration:
      - JSON Schema Example: Defines fields such as `totalDays`, `itinerary` array with days and activities including id, title, description, duration, location, and type.
    - Inputs: AI Agent raw output text
    - Outputs: Parsed structured JSON object for downstream use
    - Edge Cases:  
      - AI generates non-conforming JSON  
      - Parsing exceptions on schema mismatch

---

#### 2.3 Vector Store Search

- **Overview:** Provides the AI Agent with real-time access to an indexed knowledge base of tour packages to inform its recommendations.
- **Nodes Involved:** `Answer questions with a vector store`, `Pinecone Vector Store`, `Embeddings OpenAI1`, `OpenAI Chat Model1`
- **Node Details:**

  - **Answer questions with a vector store**
    - Type: Langchain Vector Store Tool
    - Role: Acts as a tool for the AI Agent to query Pinecone vector database.
    - Configuration:
      - Role Definition: Expert tour packaging agent with 15+ years of experience  
      - Instructions: Query Pinecone for suitable packages, respond politely with emojis  
    - Inputs: Receives queries from `OpenAI Chat Model1` and `Pinecone Vector Store` outputs
    - Outputs: Provides relevant tour package info back to the AI Agent
    - Edge Cases:  
      - Vector store unavailability or auth errors  
      - No relevant matches found  
      - Latency in vector search responses

  - **Pinecone Vector Store**
    - Type: Langchain Vector Store Pinecone Node
    - Role: Connects with Pinecone API to perform vector similarity searches.
    - Configuration:
      - Pinecone Index: `tourpackagerecommendation` (pre-built index of tours)
      - Credentials: Pinecone API key and environment
    - Inputs: Embeddings from `Embeddings OpenAI1`
    - Outputs: Vector similarity search results forwarded to `Answer questions with a vector store`
    - Edge Cases:  
      - API key invalid or expired  
      - Index not found or misconfigured

  - **Embeddings OpenAI1**
    - Type: Langchain OpenAI Embeddings Node
    - Role: Converts input texts into vector embeddings using OpenAI's `text-embedding-ada-002` model.
    - Configuration:
      - Model: text-embedding-ada-002  
      - Credentials: OpenAI API key
    - Inputs: Text inputs from AI Agent or vector store queries
    - Outputs: Embeddings for vector similarity search
    - Edge Cases:  
      - API rate limits or quota exhaustion  
      - Model deprecation or errors

  - **OpenAI Chat Model1**
    - Type: Langchain OpenAI Chat Model Node
    - Role: Supports vector store tool by generating or interpreting queries.
    - Configuration:
      - Model: gpt-4o  
      - Credentials: OpenAI API key
    - Inputs: From vector store tool
    - Outputs: Text or JSON for vector store query assistance
    - Edge Cases: Similar to other OpenAI nodes (timeouts, rate limits)

---

#### 2.4 AI Response Composition and Delivery

- **Overview:** Finalizes the tour package recommendation by integrating AI-generated content, structured formatting, and returning it to the user interface.
- **Nodes Involved:** Covered under `Tour Recommendation AI Agent` and `Respond to Webhook` nodes described above.
- **Additional Notes:**  
  - The AI Agent’s output parser ensures compliance with the front-end’s expected JSON schema.  
  - The response node guarantees the output is sent correctly to the Lovable UI.

---

### 3. Summary Table

| Node Name                  | Node Type                                     | Functional Role                                 | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                        |
|----------------------------|-----------------------------------------------|------------------------------------------------|-------------------------------|--------------------------------|------------------------------------------------------------------|
| Webhook                    | n8n-nodes-base.webhook                        | Entry point for Lovable UI POST requests       | -                             | Tour Recommendation AI Agent   | See Sticky Note5 for webhook setup and Lovable UI instructions  |
| Tour Recommendation AI Agent | @n8n/n8n-nodes-langchain.agent               | Core AI logic orchestrator                      | Webhook, OpenAI Chat Model, Answer questions with vector store, Structured Output Parser | Respond to Webhook             |                                                                    |
| OpenAI Chat Model           | @n8n/n8n-nodes-langchain.lmChatOpenAi        | Provides GPT-4o language model responses        | -                             | Tour Recommendation AI Agent   |                                                                    |
| Answer questions with a vector store | @n8n/n8n-nodes-langchain.toolVectorStore      | Queries Pinecone vector database for tours     | OpenAI Chat Model1, Pinecone Vector Store | Tour Recommendation AI Agent   |                                                                    |
| Pinecone Vector Store       | @n8n/n8n-nodes-langchain.vectorStorePinecone | Connects to Pinecone vector DB for similarity search | Embeddings OpenAI1            | Answer questions with a vector store |                                                                    |
| Embeddings OpenAI1          | @n8n/n8n-nodes-langchain.embeddingsOpenAi    | Generates text embeddings for vector search    | -                             | Pinecone Vector Store          |                                                                    |
| OpenAI Chat Model1          | @n8n/n8n-nodes-langchain.lmChatOpenAi        | Supports vector store query formulation         | -                             | Answer questions with a vector store |                                                                    |
| Structured Output Parser    | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI output into structured JSON           | Tour Recommendation AI Agent  | Tour Recommendation AI Agent   |                                                                    |
| Respond to Webhook          | n8n-nodes-base.respondToWebhook                | Sends JSON response back to Lovable UI client  | Tour Recommendation AI Agent  | -                              |                                                                    |
| Sticky Note1                | n8n-nodes-base.stickyNote                      | Documentation banner                            | -                             | -                              | ## Personalized Tour Package Recommendations connecting to Lovable UI |
| Sticky Note5                | n8n-nodes-base.stickyNote                      | Setup instructions and tech stack details       | -                             | -                              | Pre-requisite: Vectorize tours in Pinecone DB; Lovable UI integration instructions; Workflow tech stack |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: `n8n-nodes-base.webhook`  
   - Set HTTP Method: `POST`  
   - Set Path: a unique identifier (e.g., `12b44ee5-c43e-430c-a1d4-4fc5ff5e45c4`)  
   - Response Mode: `Response Node`  
   - Purpose: Receive user requests from Lovable UI form

2. **Create OpenAI Chat Model Node (Primary AI Model)**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: `gpt-4o`  
   - Credentials: OpenAI API key (ensure valid)  
   - Additional Options: Set response format to `json_object` for structured handling

3. **Create Embeddings OpenAI Node**  
   - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
   - Model: `text-embedding-ada-002`  
   - Credentials: OpenAI API key

4. **Create Pinecone Vector Store Node**  
   - Type: `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
   - Configure with:  
     - Pinecone Index: `tourpackagerecommendation` (ensure this index exists and is populated)  
     - Credentials: Pinecone API key and environment

5. **Create OpenAI Chat Model Node (Vector Store Support)**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: `gpt-4o`  
   - Credentials: OpenAI API key

6. **Create Vector Store Tool Node**  
   - Type: `@n8n/n8n-nodes-langchain.toolVectorStore`  
   - Parameters:  
     - Description: Define role as an expert travel agent with 15+ years experience  
     - Instructions to query Pinecone and respond politely with emojis  
   - Connect Inputs from `OpenAI Chat Model1` and `Pinecone Vector Store`

7. **Create Structured Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Configure JSON schema for itinerary with fields: `totalDays`, `itinerary` (days array with activities)  
   - Use sample schema provided in the workflow for reference

8. **Create AI Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - Text Input Expression: `={{ $json.body.destination }}` (captures destination from webhook request)  
     - System Message: Define as experienced travel agent using Pinecone for tours, output as per structured parser  
     - Enable Output Parser using the above `Structured Output Parser` node  
     - Prompt Type: Define  
   - Connect AI Language Model input to the primary `OpenAI Chat Model`  
   - Connect AI Tool to the `Answer questions with a vector store` node  
   - Connect AI Output Parser to `Structured Output Parser`

9. **Create Respond to Webhook Node**  
   - Type: `n8n-nodes-base.respondToWebhook`  
   - Configure to respond with JSON  
   - Response Body: Set to output the AI Agent’s parsed JSON (`={{ $json.output }}`)

10. **Connect Nodes in Order:**  
    - `Webhook` → `Tour Recommendation AI Agent`  
    - `OpenAI Chat Model` → AI Agent’s language model input  
    - `Embeddings OpenAI1` → `Pinecone Vector Store`  
    - `OpenAI Chat Model1` and `Pinecone Vector Store` → `Answer questions with a vector store`  
    - `Answer questions with a vector store` → AI Agent’s tool input  
    - `Structured Output Parser` → AI Agent’s output parser input  
    - `Tour Recommendation AI Agent` → `Respond to Webhook`

11. **Credential Setup:**  
    - OpenAI API credentials for all OpenAI nodes  
    - Pinecone API credentials for Pinecone Vector Store node

12. **Populate Pinecone Index:**  
    - Use a prior workflow or process to embed and upload tour and activity data into the `tourpackagerecommendation` Pinecone index. See linked workflow in notes.

13. **Test Workflow:**  
    - Send POST requests with JSON body containing `destination` field to the webhook URL.  
    - Validate JSON structured itinerary is returned with recommended tour packages.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Pre-requisite: Vectorize Tours and Activities information in Pinecone Vector Database using a workflow like: https://n8n.io/workflows/5085-convert-tour-pdfs-to-vector-database-using-google-drive-langchain-and-openai/                                                                                                                                                                                        | Pinecone vectorization workflow for tour data                                                           |
| UI Integration: Lovable UI is a form-based user interface that submits destination or activity queries to this n8n webhook and receives JSON responses for display.                                                                                                                                                                                                                                        | Lovable UI integration details                                                                           |
| Structured Output Parser: The AI responses are parsed to a JSON schema with itinerary details including day-wise activities, their descriptions, durations, and locations. This enables Lovable UI or other frontends to render the recommendations effectively.                                                                                                                                               | JSON schema example provided in Structured Output Parser node                                            |
| Tech Stack: OpenAI GPT-4o for AI chat, OpenAI text-embedding-ada-002 for embeddings, Pinecone as vector DB, n8n for automation, Lovable for UI                                                                                                                                                                                                                                                             | Technology components summary                                                                             |
| Workflow Description Sticky Notes: Detailed instructions, system roles, and integration tips are embedded as sticky notes within the workflow canvas for easy reference by maintainers and collaborators.                                                                                                                                                                                                | In-workflow sticky notes                                                                                 |

---

_Disclaimer: The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public._