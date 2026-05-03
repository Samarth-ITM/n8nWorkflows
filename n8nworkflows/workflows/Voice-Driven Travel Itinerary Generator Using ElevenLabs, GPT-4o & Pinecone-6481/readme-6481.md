Voice-Driven Travel Itinerary Generator Using ElevenLabs, GPT-4o & Pinecone

https://n8nworkflows.xyz/workflows/voice-driven-travel-itinerary-generator-using-elevenlabs--gpt-4o---pinecone-6481


# Voice-Driven Travel Itinerary Generator Using ElevenLabs, GPT-4o & Pinecone

### 1. Workflow Overview

This workflow, titled **"Voice-Driven Travel Itinerary Generator Using ElevenLabs, GPT-4o & Pinecone"**, is designed to provide personalized travel itinerary recommendations based on voice input. The core use case targets travel agencies or travel service providers who want to automate customer interactions via voice commands, leveraging AI to generate rich, context-aware tour packages.

The workflow is logically divided into the following blocks:

- **1.1 Voice Input Reception:** Accepts voice-based customer queries via a webhook connected to ElevenLabs voice agent.
- **1.2 AI-Powered Tour Recommendation Agent:** Processes the customer input using a GPT-4.1 mini model and a customized LangChain AI agent that references a Pinecone vector store of tours and activities to recommend personalized itineraries.
- **1.3 Vector Store Interaction:** Utilizes OpenAI embeddings to vectorize tour data and store or query this data in Pinecone, enabling semantic search and retrieval.
- **1.4 Itinerary Construction and Response:** Combines AI-generated recommendations and vector store lookups to build coherent tour packages, then responds back to the customer via the webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Voice Input Reception

- **Overview:** This block captures incoming POST requests containing customer travel queries from the ElevenLabs voice agent. It serves as the entry point for the workflow.
- **Nodes Involved:**  
  - Voice Agent Webhook  
  - Tour Recommendation AI Agent1 (input connection)  
  - Respond to Webhook (final response node)

- **Node Details:**

  - **Voice Agent Webhook**  
    - Type: Webhook (n8n-nodes-base.webhook)  
    - Role: Listens for HTTP POST requests at path `/travel`.  
    - Configuration: HTTP method POST; response mode set to "responseNode" to enable response after processing.  
    - Inputs: External voice agent (ElevenLabs) POSTs customer queries here.  
    - Outputs: Passes raw request data (JSON body) to Tour Recommendation AI Agent1.  
    - Edge cases:  
      - Invalid or malformed HTTP requests may cause failure.  
      - Timeout if downstream processing is slow.  
      - Requires the webhook URL to be correctly configured in ElevenLabs.  
    - Version: Version 2 of the webhook node.

  - **Tour Recommendation AI Agent1**  
    - Type: LangChain Agent node (`@n8n/n8n-nodes-langchain.agent`)  
    - Role: Acts as the primary AI agent to parse and process the customer query text; applies a travel agent system prompt to produce structured tour recommendations.  
    - Configuration:  
      - Input text is extracted dynamically from the webhook JSON body (`={{ $json.body }}`).  
      - System message specifies travel agent role, referencing Pinecone database as the tours source.  
      - Includes detailed tour scheduling rules (e.g., time constraints, meal defaults, activity grouping).  
      - Prompt type: "define" (likely defining the prompt behavior).  
    - Inputs: Raw customer query from webhook.  
    - Outputs: Passes AI-generated itinerary text to "Respond to Webhook" node.  
    - Edge cases:  
      - AI model errors or timeout.  
      - Incomplete or ambiguous customer input may yield poor recommendations.  
      - Dependency on Pinecone vector store data availability.  
    - Version: 1.9 (latest agent node).

  - **Respond to Webhook**  
    - Type: Respond to Webhook (n8n-nodes-base.respondToWebhook)  
    - Role: Sends the final AI-generated itinerary back to the customer as the HTTP response.  
    - Configuration: Default options (no custom headers or status codes).  
    - Inputs: Receives final text from Tour Recommendation AI Agent1.  
    - Outputs: None (end node).  
    - Edge cases:  
      - Failure if no input is received.  
      - Network issues can affect response delivery.  
    - Version: 1.1.

---

#### 2.2 AI-Powered Tour Recommendation and Query Processing

- **Overview:** This block uses OpenAI language models to refine customer queries and leverage vector search to recommend tours. It includes an intermediary Q&A node that queries Pinecone for relevant tour packages and restructures the response.

- **Nodes Involved:**  
  - OpenAI Chat Model3 (GPT-4.1 mini)  
  - Tour Recommendation AI Agent1 (also part of block 1.1)  
  - OpenAI Chat Model2 (GPT-4o)  
  - Tour Builder Q&A (Tool Vector Store node)  
  - Tour List Data store (Vector Store Pinecone node)  
  - Embeddings OpenAI  

- **Node Details:**

  - **OpenAI Chat Model3**  
    - Type: LangChain Chat Model (gpt-4.1-mini)  
    - Role: Processes input text to produce refined AI understanding or intermediate chat completions.  
    - Configuration: Model set to GPT-4.1-mini.  
    - Inputs: Receives data from external nodes, feeding into Tour Recommendation AI Agent1.  
    - Outputs: Passes processed text to Tour Recommendation AI Agent1.  
    - Edge cases: API rate limits, timeouts, or invalid input could cause errors.  
    - Version: 1.2.

  - **OpenAI Chat Model2**  
    - Type: LangChain Chat Model (gpt-4o)  
    - Role: Provides advanced language modeling for the Q&A about tours.  
    - Configuration: Model set to GPT-4o (latest version).  
    - Inputs: Receives input from Tour Builder Q&A node.  
    - Outputs: Passes completion results back to Tour Builder Q&A.  
    - Edge cases: Similar to above; API limits, model errors.  
    - Version: 1.2.

  - **Tour Builder Q&A**  
    - Type: LangChain Tool Vector Store (`@n8n/n8n-nodes-langchain.toolVectorStore`)  
    - Role: Acts as an expert travel agent querying the Pinecone vector store to find suitable tour packages based on customer needs.  
    - Configuration:  
      - Role description emphasizes 15+ years experience in tour packaging and polite, friendly responses with emojis.  
      - Queries Pinecone vector store for matching packages and reconstructs best fits.  
    - Inputs: Receives AI language model outputs and vector store data.  
    - Outputs: Provides tour package recommendations to OpenAI Chat Model2.  
    - Edge cases:  
      - No matching vector results may reduce recommendation quality.  
      - Vector store connectivity issues.  
      - Expression errors in querying.  
    - Version: 1.1.

  - **Tour List Data store**  
    - Type: Vector Store Pinecone node (`@n8n/n8n-nodes-langchain.vectorStorePinecone`)  
    - Role: Interfaces with Pinecone vector database to store or query embedded tour data.  
    - Configuration: Uses Pinecone index (configured via credentials).  
    - Inputs: Receives embeddings from Embeddings OpenAI node.  
    - Outputs: Provides vector search results to Tour Builder Q&A node.  
    - Edge cases:  
      - Pinecone API key expiry or misconfiguration.  
      - Network or index availability issues.  
    - Version: 1.1.

  - **Embeddings OpenAI**  
    - Type: LangChain Embeddings OpenAI node  
    - Role: Generates vector embeddings for tour-related textual data using OpenAI's `text-embedding-ada-002` model.  
    - Configuration: Model set to `text-embedding-ada-002`.  
    - Inputs: Receives raw tour text data (external or internal source not shown in this workflow).  
    - Outputs: Passes embeddings to Pinecone vector store node.  
    - Edge cases: API limits, input text too large, or malformed data.  
    - Version: 1.2.

---

#### 2.3 General Notes and Visual Aids

- **Sticky Note2**  
  - Content: "## Voice Driven Travel Agent" (Title block for clarity)  
  - Position: Near the webhook and AI agent nodes grouping.

- **Sticky Note3**  
  - Content:  
    ```
    ## Steps
    1. Add the Voice Agent Webhook: Connect to Elevenlabs Voice Agent by copying and configuring the webhook URL in Elevenlabs using the POST method.
    Format the Tour query from the customer breaking it down to Destination, Type of your, Number of days and Number of Passengers.

    2. Add Tour Recomendation AI Agent: Give appropiate System Prompt and the references to the Tools available.

    3. OpenaAI Model: ChatGPT-4o

    4. Simple memory - upto 5 context window length

    5. Answer Questions with Vector Store: Add specific role and instructions to extract the tour packages from the pinecone database.

    6. Connect the Vectorized Tours and Activities content and store in pinecone database. Refer to the link below to create embedded content and store it in the vector detabase
    (https://creators.n8n.io/workflows/5085)

    8. Respospond to Webhook: Respond to the customer tour request with itinerary plan.
    ```
  - Provides a high-level step guide and a link to a related n8n workflow for vector database setup.

---

### 3. Summary Table

| Node Name                | Node Type                                | Functional Role                                  | Input Node(s)                  | Output Node(s)               | Sticky Note                                                  |
|--------------------------|----------------------------------------|-------------------------------------------------|-------------------------------|-----------------------------|--------------------------------------------------------------|
| Voice Agent Webhook       | Webhook (n8n-nodes-base.webhook)       | Receives voice-based customer travel queries    | External (ElevenLabs voice)    | Tour Recommendation AI Agent1 | ## Voice Driven Travel Agent                                  |
| Tour Recommendation AI Agent1 | LangChain Agent (@n8n/n8n-nodes-langchain.agent) | AI agent to generate structured tour itineraries | Voice Agent Webhook, OpenAI Chat Model3 | Respond to Webhook             | ## Voice Driven Travel Agent                                  |
| Respond to Webhook        | Respond to Webhook (n8n-nodes-base.respondToWebhook) | Sends final itinerary response to customer      | Tour Recommendation AI Agent1  | None                        | ## Voice Driven Travel Agent                                  |
| OpenAI Chat Model3        | LangChain Chat Model (GPT-4.1 mini)   | AI chat model for intermediate text processing  | None (feeds Tour Recommendation AI Agent1) | Tour Recommendation AI Agent1 |                                                              |
| OpenAI Chat Model2        | LangChain Chat Model (GPT-4o)          | Advanced AI model for tour Q&A                   | Tour Builder Q&A               | Tour Builder Q&A            |                                                              |
| Tour Builder Q&A          | LangChain Tool Vector Store            | Queries Pinecone store and builds tour packages | OpenAI Chat Model2, Tour List Data store | OpenAI Chat Model2           |                                                              |
| Tour List Data store      | Vector Store Pinecone                   | Interfaces with Pinecone vector DB               | Embeddings OpenAI             | Tour Builder Q&A            |                                                              |
| Embeddings OpenAI         | LangChain Embeddings OpenAI             | Creates vector embeddings for tour data          | External textual tour data (not shown) | Tour List Data store         |                                                              |
| Sticky Note2              | Sticky Note                            | Visual title block                                | None                          | None                        | ## Voice Driven Travel Agent                                  |
| Sticky Note3              | Sticky Note                            | Workflow steps and instructions                   | None                          | None                        | See detailed steps and link: https://creators.n8n.io/workflows/5085 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (Voice Agent Webhook)**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `travel`  
   - Response Mode: `responseNode`  
   - Save and copy the webhook URL for use in ElevenLabs voice agent configuration.

2. **Create LangChain Agent Node (Tour Recommendation AI Agent1)**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - Text input: `={{ $json.body }}` (extract query text from webhook payload)  
     - System Message: Define as an experienced travel agent who builds tour packages referencing Pinecone vector database. Include detailed scheduling and grouping rules (see Section 2.1).  
     - Prompt type: `define`  
   - Connect input from "Voice Agent Webhook" main output.

3. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - No special parameters needed.  
   - Connect input from "Tour Recommendation AI Agent1" node output.

4. **Create OpenAI Chat Model Node (OpenAI Chat Model3)**  
   - Type: LangChain Chat Model  
   - Model: `gpt-4.1-mini`  
   - Connect output to input of "Tour Recommendation AI Agent1" node (for intermediate processing).

5. **Create OpenAI Chat Model Node (OpenAI Chat Model2)**  
   - Type: LangChain Chat Model  
   - Model: `gpt-4o`  
   - Connect outputs to and from "Tour Builder Q&A" node to enable advanced Q&A processing.

6. **Create Tour Builder Q&A Node (Tool Vector Store)**  
   - Type: LangChain Tool Vector Store  
   - Description: Define role as expert tour packaging agent with 15+ years experience. Instructions to query Pinecone vector database for best packages. Friendly tone with emojis.  
   - Connect input from "OpenAI Chat Model2" and output back to it.

7. **Create Tour List Data store Node (Vector Store Pinecone)**  
   - Type: Vector Store Pinecone  
   - Configure Pinecone index credentials (set up Pinecone API key and index name).  
   - Connect input from "Embeddings OpenAI" and output to "Tour Builder Q&A".

8. **Create Embeddings OpenAI Node**  
   - Type: LangChain Embeddings OpenAI  
   - Model: `text-embedding-ada-002`  
   - Input: Tour text data (to be prepared externally or from another workflow).  
   - Output: Connect to "Tour List Data store" node.

9. **Add Sticky Notes for Documentation**  
   - Sticky Note2 near webhook and AI agent: Content `## Voice Driven Travel Agent`  
   - Sticky Note3 with workflow steps and link: Place prominently for user reference.

10. **Credential Setup**  
    - OpenAI API credentials configured for all OpenAI nodes.  
    - Pinecone API credentials configured for vector store node.

11. **Testing and Validation**  
    - Configure ElevenLabs voice agent to POST voice query JSON to the webhook URL.  
    - Test various travel queries and validate itinerary responses.  
    - Monitor for errors such as API failures or vector store disconnections.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                           |
|----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow integrates ElevenLabs voice agent via webhook for voice input reception.                                            | ElevenLabs voice agent webhook setup                      |
| The Pinecone vector database stores embedded tour and activity data enabling semantic search for personalized recommendations.   | https://creators.n8n.io/workflows/5085                     |
| The AI agent uses GPT-4o and GPT-4.1-mini models for natural language understanding and generation, requiring valid OpenAI API keys. | OpenAI API usage and rate limits                          |
| The tour recommendation system enforces strict scheduling rules to generate realistic daily itineraries with meal defaults.    | See system prompt details in Tour Recommendation AI Agent1 |
| The workflow uses a simple memory window (~5 messages) to maintain context during interactions.                                  | See Sticky Note3 content                                   |
| For best results, ensure Pinecone index is populated with high-quality embedded tour data before running real customer queries. | Pinecone vector store initialization                       |

---

This completes the comprehensive reference documentation for the **Voice-Driven Travel Itinerary Generator Using ElevenLabs, GPT-4o & Pinecone** workflow.