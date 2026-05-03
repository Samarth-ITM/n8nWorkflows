Build Your First AI Agent

https://n8nworkflows.xyz/workflows/build-your-first-ai-agent-6270


# Build Your First AI Agent

### 1. Workflow Overview

This workflow, titled **Build Your First AI Agent**, is designed to create a conversational AI assistant using n8n’s AI agent capabilities. It targets users who want to deploy an interactive chatbot that leverages real-time data and external services to answer questions and perform tasks automatically. The AI agent integrates Google Gemini as its language model and uses various “tools” (like weather and news fetchers) to provide dynamic, context-aware responses.

The workflow is logically divided into the following blocks:

- **1.1 User Interaction**: Captures user messages through a chat interface exposed via webhook.
- **1.2 AI Model Connection**: Connects to Google Gemini for natural language processing.
- **1.3 Memory Management**: Maintains recent conversation context to keep interactions coherent.
- **1.4 AI Agent Core Logic**: The central node orchestrating the agent’s decision-making, tool usage, and response formulation.
- **1.5 External Tools**: Nodes providing real-world data inputs (weather and news) callable by the AI agent.
- **1.6 Documentation & User Guidance**: Multiple sticky notes providing instructions, credentials setup guidance, and usage tips.

---

### 2. Block-by-Block Analysis

#### 2.1 User Interaction Block

- **Overview:**  
  This block provides the entry point for users to send messages to the AI agent. It handles the incoming chat messages via a webhook and sends them forward for processing.

- **Nodes Involved:**  
  - `Example Chat`

- **Node Details:**  
  - **Example Chat**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Captures user chat input from a public-facing webhook URL.  
    - Configuration:  
      - Public webhook enabled, titled “Your first AI Agent 🚀” with a subtitle and placeholder text for the input field.  
      - Initial greeting message: “Hi there! 👋”  
      - Response mode set to use the output of the last node in the chain.  
    - Inputs: None (webhook trigger)  
    - Outputs: Connected to `Your First AI Agent` node.  
    - Edge Cases:  
      - Network issues or webhook failures could prevent message reception.  
      - User inputs outside expected formats might cause parsing issues downstream.

---

#### 2.2 AI Model Connection Block

- **Overview:**  
  Connects the workflow to Google Gemini, the language model powering the AI agent’s natural language understanding and generation.

- **Nodes Involved:**  
  - `Connect your model`

- **Node Details:**  
  - **Connect your model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
    - Role: Sends prompts and receives AI-generated completions from Google Gemini.  
    - Configuration:  
      - Temperature set to 0 for deterministic outputs.  
      - Requires Google AI API key credential (free key from Google AI Studio).  
    - Inputs: Connected from `Your First AI Agent` as its language model provider.  
    - Outputs: Connected back to `Your First AI Agent`.  
    - Edge Cases:  
      - Authentication or quota errors if API key is invalid or exhausted.  
      - API timeouts or rate limiting from Google Gemini service.  
    - Notes:  
      - Credential setup instructions provided in sticky notes.

---

#### 2.3 Memory Management Block

- **Overview:**  
  Maintains a window of recent conversation messages, enabling the AI agent to keep context and provide coherent, context-aware answers.

- **Nodes Involved:**  
  - `Conversation Memory`

- **Node Details:**  
  - **Conversation Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Stores the last 30 messages in the chat to keep conversation context.  
    - Configuration:  
      - `Context Window Length` set to 30 messages (adjustable).  
    - Inputs: Connected to `Your First AI Agent` node’s memory input.  
    - Outputs: Feeds memory to `Your First AI Agent`.  
    - Edge Cases:  
      - Excessive conversation length beyond memory window may lead to context loss.  
      - Memory node failures would cause the agent to lose conversation state.

---

#### 2.4 AI Agent Core Logic Block

- **Overview:**  
  The central orchestrator of the workflow — this agent node receives user input, accesses memory, calls tools as needed, and formulates final responses.

- **Nodes Involved:**  
  - `Your First AI Agent`

- **Node Details:**  
  - **Your First AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Implements AI agent logic with defined system message, tool selection, and conversational output.  
    - Configuration:  
      - System Message defines personality as a friendly, educational demo assistant.  
      - Instructions specify goal to demonstrate AI agent capabilities, interact conversationally, choose appropriate tools when needed, and present clear responses.  
      - Context includes current date/time injected dynamically (`{{ $now }}`).  
      - Output format ensures friendly, conversational replies with proactive suggestions.  
    - Inputs: Receives user input from `Example Chat`, memory from `Conversation Memory`, language model from `Connect your model`, and tool outputs from external tools (`Get Weather`, `Get News`).  
    - Outputs: Sends responses back to chat trigger node.  
    - Edge Cases:  
      - Misinterpretation of user input may lead to wrong tool selection.  
      - Tool call failures or unexpected tool outputs can disrupt response quality.  
      - Complex or multi-step user requests may exceed single tool usage capability.  
    - Notes:  
      - “Tool” input can be dynamically extended with additional tools like Gmail or Calendar.  
      - Sticky notes explain the agent’s architecture and encourage system message tweaking.

---

#### 2.5 External Tools Block

- **Overview:**  
  Provides real-time data fetching capabilities for the AI agent to use. Includes weather forecasting and news retrieval.

- **Nodes Involved:**  
  - `Get Weather`  
  - `Get News`

- **Node Details:**  
  - **Get Weather**  
    - Type: `n8n-nodes-base.httpRequestTool`  
    - Role: Calls Open-Meteo API to fetch weather forecast data.  
    - Configuration:  
      - URL: `https://api.open-meteo.com/v1/forecast`  
      - Query parameters include latitude, longitude, requested weather variables (current, hourly, daily), date range, and temperature units.  
      - Parameters are dynamically inferred or overridden by AI expressions to avoid asking the user explicitly.  
    - Inputs: Called by `Your First AI Agent` when weather information is requested.  
    - Outputs: Weather data passed back to agent for response generation.  
    - Edge Cases:  
      - Invalid or missing location data may cause API errors.  
      - Network/API failures or quota issues possible.  
      - Date parameter inconsistencies (start_date after end_date) must be avoided.  

  - **Get News**  
    - Type: `n8n-nodes-base.rssFeedReadTool`  
    - Role: Retrieves latest news articles from selected RSS feeds.  
    - Configuration:  
      - URL is selected dynamically from an AI-curated list of popular news feeds (BBC, Al Jazeera, CNN, TechCrunch, Hacker News, n8n Blog, etc.).  
    - Inputs: Invoked by `Your First AI Agent` when news updates are requested.  
    - Outputs: Latest news items for agent to summarize or present.  
    - Edge Cases:  
      - RSS feed availability or format changes may break parsing.  
      - Network errors or slow responses may delay replies.

---

#### 2.6 Documentation & User Guidance Block

- **Overview:**  
  Provides comprehensive user guidance, credentials setup instructions, interactive tips, and encouragement notes via sticky notes scattered throughout the workflow.

- **Nodes Involved:**  
  - `Introduction Note`  
  - `Sticky Note12`  
  - `Sticky Note13`  
  - `Sticky Note15`  
  - `Sticky Note16`  
  - `Sticky Note17`  
  - `Sticky Note1`  
  - `Sticky Note`

- **Node Details:**  
  - All nodes are `n8n-nodes-base.stickyNote` type.  
  - Contain branding, detailed setup instructions for Google Gemini API key acquisition and credential creation.  
  - Provide example user queries for the chat interface.  
  - Explain the agent’s architecture and tool usage philosophy.  
  - Encourage adding more tools (like Gmail or Google Calendar) by extending the agent’s tool input.  
  - Provide useful links:  
    - Coaching booking: https://api.ia2s.app/form/templates/coaching?template=Very%20First%20AI%20Agent  
    - Feedback submission: https://api.ia2s.app/form/templates/feedback?template=Very%20First%20AI%20Agent  
    - Google AI Studio key creation: https://aistudio.google.com/app/apikey  
  - Positioned for easy discovery near relevant functional nodes.  
  - Edge Cases: None (informational only).

---

### 3. Summary Table

| Node Name           | Node Type                                  | Functional Role                | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                                          |
|---------------------|--------------------------------------------|-------------------------------|---------------------|---------------------|----------------------------------------------------------------------------------------------------------------------|
| Introduction Note    | Sticky Note                                | Workflow introduction & branding | None                | None                | © 2025 Lucas Peyrin                                                                                                  |
| Sticky Note12       | Sticky Note                                | Chat interface usage tips       | None                | None                | © 2025 Lucas Peyrin                                                                                                  |
| Sticky Note13       | Sticky Note                                | AI Agent explanation            | None                | None                | © 2025 Lucas Peyrin                                                                                                  |
| Sticky Note15       | Sticky Note                                | Memory node explanation         | None                | None                | © 2025 Lucas Peyrin                                                                                                  |
| Sticky Note16       | Sticky Note                                | External tools explanation      | None                | None                | © 2025 Lucas Peyrin                                                                                                  |
| Sticky Note17       | Sticky Note                                | Gemini API credential setup     | None                | None                | © 2025 Lucas Peyrin                                                                                                  |
| Sticky Note1        | Sticky Note                                | Adding more tools guidance      | None                | None                | © 2025 Lucas Peyrin                                                                                                  |
| Sticky Note         | Sticky Note                                | Sharing public chat URL tip     | None                | None                | © 2025 Lucas Peyrin                                                                                                  |
| Example Chat        | Chat Trigger                              | Captures user chat input        | None                | Your First AI Agent  | © 2025 Lucas Peyrin                                                                                                  |
| Connect your model  | Langchain Google Gemini LM Node           | Connects to Google Gemini model | Your First AI Agent  | Your First AI Agent  | © 2025 Lucas Peyrin                                                                                                  |
| Conversation Memory | Langchain Memory Buffer Window            | Maintains chat context          | Your First AI Agent  | Your First AI Agent  | © 2025 Lucas Peyrin                                                                                                  |
| Your First AI Agent | Langchain Agent Node                       | Core AI agent logic             | Example Chat, Connect your model, Conversation Memory, Get Weather, Get News | Example Chat      | © 2025 Lucas Peyrin                                                                                                  |
| Get Weather         | HTTP Request Tool                         | Fetches weather data            | Your First AI Agent  | Your First AI Agent  | © 2025 Lucas Peyrin                                                                                                  |
| Get News            | RSS Feed Read Tool                        | Fetches latest news articles    | Your First AI Agent  | Your First AI Agent  | © 2025 Lucas Peyrin                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation**  
   - Create multiple sticky note nodes to document instructions, usage tips, and credential setup:  
     - Introduction Note with workflow purpose and coaching/feedback links.  
     - Sticky Note12 with chat interface usage examples.  
     - Sticky Note13 explaining AI Agent architecture.  
     - Sticky Note15 detailing memory node purpose.  
     - Sticky Note16 describing external tools usage.  
     - Sticky Note17 guiding Google Gemini API key creation and credential setup.  
     - Sticky Note1 suggesting how to add more tools.  
     - Sticky Note with tip on sharing public chat URL.

2. **Set Up the Chat Trigger Node**  
   - Add a `@n8n/n8n-nodes-langchain.chatTrigger` node named `Example Chat`.  
   - Configure as public webhook with title “Your first AI Agent 🚀”, subtitle, input placeholder, no welcome screen.  
   - Add initial messages: “Hi there! 👋”.

3. **Configure the AI Language Model Node**  
   - Add `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` node named `Connect your model`.  
   - Set temperature parameter to 0.  
   - Create and assign Google AI API key credential by:  
     - Visiting https://aistudio.google.com/app/apikey  
     - Creating a new API key and adding it as a credential in n8n.

4. **Add Memory Buffer Node**  
   - Add `@n8n/n8n-nodes-langchain.memoryBufferWindow` node named `Conversation Memory`.  
   - Set `Context Window Length` to 30 messages.

5. **Create the AI Agent Node**  
   - Add `@n8n/n8n-nodes-langchain.agent` node named `Your First AI Agent`.  
   - Configure the system message with personality, instructions, goals, context (including `{{ $now }}`), and output format as per the overview section.  
   - Connect inputs:  
     - `Example Chat` → main input (user messages)  
     - `Connect your model` → languageModel input  
     - `Conversation Memory` → ai_memory input  
     - Add tool inputs from external tools (weather and news nodes).  
   - Set output to connect back to `Example Chat` for final responses.

6. **Add External Tool Nodes**  
   - Add `n8n-nodes-base.httpRequestTool` named `Get Weather`:  
     - URL: `https://api.open-meteo.com/v1/forecast`  
     - Set query parameters: latitude, longitude, current, hourly, daily, start_date, end_date, temperature_unit.  
     - Use expressions or AI overrides to infer values automatically from user input or defaults.  
   - Add `n8n-nodes-base.rssFeedReadTool` named `Get News`:  
     - Configure URL parameter to allow AI to select from a list of known RSS feeds dynamically.  
     - No authentication required.

7. **Connect All Nodes**  
   - Link outputs from `Example Chat` to `Your First AI Agent`.  
   - Link `Connect your model` as the language model input to `Your First AI Agent`.  
   - Link `Conversation Memory` as memory input to `Your First AI Agent`.  
   - Link `Get Weather` and `Get News` as tools inputs to `Your First AI Agent`.  
   - Connect `Your First AI Agent` output back to `Example Chat` for response delivery.

8. **Test and Activate Workflow**  
   - Test by sending chat messages to the webhook URL.  
   - Observe the AI agent’s ability to call weather and news tools.  
   - Activate workflow to allow public usage.  
   - Optionally, share the chat URL for external users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Coaching sessions to level up n8n skills available.                                                                                      | https://api.ia2s.app/form/templates/coaching?template=Very%20First%20AI%20Agent                                      |
| Feedback can be submitted to help improve this AI agent template.                                                                        | https://api.ia2s.app/form/templates/feedback?template=Very%20First%20AI%20Agent                                       |
| Google AI Studio API keys needed for Gemini integration; free signup and key creation instructions provided.                             | https://aistudio.google.com/app/apikey                                                                                |
| Suggest to keep AI agent toolset focused (under 10-15 tools) for reliability and clarity.                                                | Included in system message and sticky notes                                                                           |
| For complex backend processes, structured workflows recommended over AI agent logic for reliability.                                     | Included in system message                                                                                             |
| Example user prompts to try: “What’s the weather in Paris?”, “Get me the latest tech news.”                                              | Sticky Note12                                                                                                         |
| The workflow is designed for demonstration and educational purposes by Lucas Peyrin, © 2025.                                            | Branding included in all sticky notes                                                                                  |
| To extend functionality, add nodes for Gmail or Google Calendar and connect as tools to the AI agent node’s `Tool` input.                | Sticky Note1                                                                                                          |
| The AI agent uses a simple memory window (30 messages) to stay on topic; adjust as needed to balance context and performance.           | Sticky Note15                                                                                                         |

---

**Disclaimer:**  
The text provided is derived exclusively from an n8n automated workflow. All content complies with current content policies and contains no illegal, offensive, or protected elements. Only legal and public data are processed.