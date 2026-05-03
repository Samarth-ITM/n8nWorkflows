Search and Compare Flights with DeepSeek AI and Google Flights API

https://n8nworkflows.xyz/workflows/search-and-compare-flights-with-deepseek-ai-and-google-flights-api-7322


# Search and Compare Flights with DeepSeek AI and Google Flights API

---

### 1. Workflow Overview

This workflow implements a conversational AI travel assistant designed to help users search and compare flights using DeepSeek AI and Google Flights data via SerpAPI. It guides users through a step-by-step dialogue to collect essential flight search parameters such as departure and arrival airports, dates, and trip type. Once all required information is gathered, it queries real-time flight options and summarizes the top results, offering booking links upon user selection.

**Target Use Cases:**  
- Interactive flight search and booking assistance via chat  
- Real-time flight data retrieval and comparison  
- Handling both one-way and round-trip flight queries  

**Logical Blocks:**  
- **1.1 Chat Input Reception:** Receives user messages and triggers the workflow.  
- **1.2 AI Conversational Agent:** Manages dialogue flow, data collection, validation, and decision-making.  
- **1.3 AI Language Model Backend:** Provides natural language understanding and generation via DeepSeek chat model.  
- **1.4 Memory Management:** Maintains conversational context and history to provide coherent multi-turn dialogue.  
- **1.5 Flight Search Tool Integration:** Executes flight search queries on Google Flights through SerpAPI and returns structured results.  

---

### 2. Block-by-Block Analysis

#### 1.1 Chat Input Reception

**Overview:**  
This block listens for incoming chat messages from users and acts as the starting point of the workflow.

**Nodes Involved:**  
- Chat

**Node Details:**  
- **Chat**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Webhook listener for chat messages; triggers the workflow upon receiving input.  
  - Configuration:  
    - Mode: webhook  
    - Public access enabled (allows external access)  
    - Webhook ID uniquely identifies this endpoint.  
  - Inputs: External HTTP requests (user chat messages)  
  - Outputs: Sends data to the AI Agent node  
  - Edge Cases: Webhook downtime, invalid or empty user input, concurrent requests.  

---

#### 1.2 AI Conversational Agent

**Overview:**  
Central dialogue manager that orchestrates the conversation, ensuring all required flight parameters are collected before making a search request. It also formats and presents flight search results and handles booking link delivery.

**Nodes Involved:**  
- AI Agent

**Node Details:**  
- **AI Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Manages conversation logic and calls attached tools (SerpAPI for flights).  
  - Configuration:  
    - System Message: Detailed instructions specifying required data fields (`departure_id`, `arrival_id`, `outbound_date`, `type`, and optionally `return_date`), behavior for missing data, formatting of results, and conversational tone.  
    - Uses current date/time dynamically for context.  
  - Inputs:  
    - Chat input from the Chat node  
    - Context/memory data from Simple Memory  
    - Language model from DeepSeek Chat Model  
    - Flight search tool results from Google_flights search in SerpApi  
  - Outputs: Replies back to the chat interface, invokes flight search tool.  
  - Edge Cases: Missing or ambiguous user data, flight search failures, empty search results, unexpected user requests (e.g., booking selection).  
  - Version: 2 (latest agent version)  

---

#### 1.3 AI Language Model Backend

**Overview:**  
Provides the conversational AI's natural language understanding and generation capabilities using DeepSeek's chat model API.

**Nodes Involved:**  
- DeepSeek Chat Model

**Node Details:**  
- **DeepSeek Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatDeepSeek`  
  - Role: Executes AI chat completions based on prompts and context.  
  - Configuration: Uses DeepSeek API credentials to authenticate and process chat messages.  
  - Inputs: From AI Agent node as language model provider.  
  - Outputs: Generated AI responses to AI Agent.  
  - Edge Cases: API authentication failure, rate limits, latency, malformed prompts.  
  - Credentials: Requires valid DeepSeek API key.  

---

#### 1.4 Memory Management

**Overview:**  
Maintains a contextual buffer of recent conversation turns to enable coherent multi-turn interactions, remembering user-provided parameters and dialogue history.

**Nodes Involved:**  
- Simple Memory

**Node Details:**  
- **Simple Memory**  
  - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - Role: Stores last 20 conversation messages to provide context to the AI Agent.  
  - Configuration: Context window length set to 20 messages.  
  - Inputs: Captures conversation data to maintain memory buffer.  
  - Outputs: Supplies memory context to AI Agent.  
  - Edge Cases: Memory overflow, context truncation leading to loss of earlier conversation details.  
  - Version: 1.3  

---

#### 1.5 Flight Search Tool Integration

**Overview:**  
Performs live flight searches using Google Flights data accessed via SerpAPI, based on parameters collected by the AI Agent.

**Nodes Involved:**  
- Google_flights search in SerpApi

**Node Details:**  
- **Google_flights search in SerpApi**  
  - Type: `n8n-nodes-serpapi.serpApiTool`  
  - Role: Queries Google Flights via SerpAPI with collected parameters (`departure_id`, `arrival_id`, `outbound_date`, `return_date`, `type`).  
  - Configuration:  
    - Operation: `google_flights`  
    - Input parameters dynamically taken from AI Agent's parsed variables via `$fromAI` expressions.  
    - Tool description: “Search for live flights from one city to another using Google Flights data”  
  - Inputs: Triggered by AI Agent when all required data fields are available.  
  - Outputs: Flight search results passed back to AI Agent for summarization and response.  
  - Edge Cases: API quota limits, incorrect or missing parameters, no flights found, network errors.  
  - Credentials: Requires SerpAPI credentials with Google Flights access.  

---

### 3. Summary Table

| Node Name                     | Node Type                                    | Functional Role                    | Input Node(s)        | Output Node(s)       | Sticky Note                          |
|-------------------------------|----------------------------------------------|----------------------------------|----------------------|----------------------|------------------------------------|
| Chat                          | @n8n/n8n-nodes-langchain.chatTrigger         | Receives user chat input          | External HTTP request | AI Agent             |                                    |
| AI Agent                      | @n8n/n8n-nodes-langchain.agent                | Core conversation management      | Chat, Simple Memory, DeepSeek Chat Model, Google_flights search in SerpApi | Chat (response)       | Detailed system prompt guides user |  
| Simple Memory                 | @n8n/n8n-nodes-langchain.memoryBufferWindow  | Maintains conversation context   | Conversation data     | AI Agent             |                                    |
| DeepSeek Chat Model           | @n8n/n8n-nodes-langchain.lmChatDeepSeek       | Provides AI language generation   | AI Agent (languageModel) | AI Agent            |                                    |
| Google_flights search in SerpApi | n8n-nodes-serpapi.serpApiTool               | Performs flight search queries    | AI Agent (tool)       | AI Agent             |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure mode as `webhook`, enable public access.  
   - Generate and note webhook URL for external chat messages.  

2. **Create AI Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent` (version 2)  
   - Set system message with detailed instructions to:  
     - Collect required flight info (`departure_id`, `arrival_id`, `outbound_date`, `type`, optionally `return_date`).  
     - Validate and request missing info politely.  
     - Call SerpAPI tool after data collection.  
     - Format and summarize top 3 flights.  
     - Provide booking links upon user selection.  
     - Maintain friendly, professional tone.  
   - Enable dynamic current date/time usage in system message (`{{ $now }}`).  

3. **Create Simple Memory Node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow` (version 1.3)  
   - Set `contextWindowLength` to 20 to buffer recent conversation turns.  

4. **Create DeepSeek Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatDeepSeek` (version 1)  
   - Set credentials using valid DeepSeek API key.  
   - No special parameter configuration needed.  

5. **Create SerpAPI Flight Search Node**  
   - Type: `n8n-nodes-serpapi.serpApiTool` (version 1)  
   - Operation: `google_flights`  
   - Configure input parameters as expressions referencing AI Agent variables:  
     - `departure_id` = `$fromAI('Departure_airport_code___location_kgmid__departure_id_', '', 'string')`  
     - `arrival_id` = `$fromAI('Arrival_airport_code___location_kgmid__arrival_id_', '', 'string')`  
     - `outbound_date` = `$fromAI('Outbound_Date__outbound_date_', '', 'string')`  
     - `return_date` = `$fromAI('Return_Date__return_date_', '', 'string')` (optional)  
   - Add tool description: "Search for live flights from one city to another using Google Flights data"  
   - Set credentials with SerpAPI account having Google Flights access.  

6. **Connect Nodes**  
   - Link `Chat` node output to `AI Agent` input.  
   - Connect `Simple Memory` output to `AI Agent` memory input.  
   - Connect `DeepSeek Chat Model` output to `AI Agent` language model input.  
   - Connect `Google_flights search in SerpApi` output to `AI Agent` tool input.  

7. **Test Workflow**  
   - Send chat messages to webhook URL.  
   - Confirm AI Agent collects data, queries flights, and replies with top options.  
   - Validate booking link responses for selected flights.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The system message in AI Agent node is critical for guiding the conversation flow and must be carefully preserved or adapted.       | System prompt inside AI Agent node parameters       |
| SerpAPI requires a valid paid account for Google Flights API access; ensure quota and billing are sufficient.                       | https://serpapi.com/google-flights-api              |
| DeepSeek AI credentials must be active and correctly configured for language model calls to work.                                    | DeepSeek API dashboard                               |
| Workflow assumes user inputs IATA airport codes; consider enhancing with validation nodes or external API for airport code lookup. | Possible future enhancement                           |
| The workflow supports both one-way and round-trip flights; return date is requested conditionally based on trip type.               | System prompt logic                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---