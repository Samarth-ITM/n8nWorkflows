Create a Webhook-Ready Conversational Assistant with Google Gemini and Session Memory

https://n8nworkflows.xyz/workflows/create-a-webhook-ready-conversational-assistant-with-google-gemini-and-session-memory-5426


# Create a Webhook-Ready Conversational Assistant with Google Gemini and Session Memory

### 1. Workflow Overview

This workflow implements a universal conversational AI assistant designed to receive user messages via a webhook and respond with context-aware, AI-generated replies powered by Google Gemini. It is optimized for integration across multiple platforms such as websites, mobile apps, chatbots, and custom dashboards.

The workflow’s logic is divided into four main functional blocks:

- **1.1 Webhook Input Reception:** Accepts incoming POST requests containing user messages and session identifiers.
- **1.2 Session Memory Management:** Maintains conversation context per user session to enable natural, contextual dialogue.
- **1.3 AI Processing Core:** Utilizes Google Gemini 2.0 Flash model with LangChain agent for generating accurate, friendly, and concise answers.
- **1.4 Response Output:** Sends back structured, JSON-formatted AI responses ready for frontend or platform consumption.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Input Reception

**Overview:**  
This block receives incoming user requests from various channels, expecting a JSON payload with a user message and a unique session ID. It acts as the entry point for the conversation.

**Nodes Involved:**  
- Webhook  
- Sticky Note (Webhook Entry Point)

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook Trigger  
  - Role: Listens for POST requests at path `/45cec96e-962d-4e27-ab75-34d25b837032`  
  - Configuration:  
    - HTTP Method: POST  
    - Response Mode: Response node (defers response until downstream node executes)  
    - Expected Payload: JSON with fields `"message"` (user input) and `"sessionId"` (unique session identifier)  
  - Inputs: External HTTP client  
  - Outputs: AI Agent node  
  - Edge Cases:  
    - Missing or malformed JSON payload  
    - Missing sessionId or message fields  
    - HTTP errors or timeouts from clients  
  - Notes: Designed for flexible integration across multiple platforms (website widgets, apps, third-party)

- **Sticky Note (Webhook Entry Point)**  
  - Content: Explains expected payload and typical sources of requests  
  - Purpose: Documentation and onboarding aid within the workflow editor

---

#### 1.2 Session Memory Management

**Overview:**  
Maintains a windowed buffer of conversation history per session, enabling the AI to reference past messages for context-aware replies.

**Nodes Involved:**  
- Simple Memory  
- Sticky Note2 (Session Memory)

**Node Details:**

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Stores recent conversation snippets to maintain context  
  - Configuration:  
    - Session Key: Extracted dynamically from incoming webhook `sessionId` field (`={{ $json.body.sessionId }}`)  
    - Session ID Type: Custom key based on user input to separate memory per user/session  
  - Inputs: None directly, but linked as AI Agent's memory source  
  - Outputs: AI Agent node (as memory source)  
  - Edge Cases:  
    - Missing sessionId leads to memory isolation failure  
    - Memory size limits could truncate important context  
  - Notes: Auto-manages conversation history enabling natural follow-up questions

- **Sticky Note2 (Session Memory)**  
  - Content: Summarizes memory features and benefits  
  - Purpose: Internal documentation

---

#### 1.3 AI Processing Core

**Overview:**  
Central AI node that interprets user messages using a defined prompt and context, generates relevant responses with Google Gemini, and integrates memory to maintain conversation flow.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model  
- Sticky Note1 (AI Processing Core)

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Orchestrates prompt definition, memory, and language model to produce AI responses  
  - Configuration:  
    - Prompt: Defines the assistant as helpful, accurate, concise, friendly, and professional  
    - Prompt Type: Define (custom prompt text)  
    - Options: Default  
    - Memory Input: Connected from Simple Memory node  
    - Language Model Input: Connected from Google Gemini Chat Model node  
  - Inputs:  
    - Main input from Webhook node (user message and session context)  
    - ai_memory input from Simple Memory  
    - ai_languageModel input from Google Gemini Chat Model  
  - Outputs: Respond to Webhook node  
  - Edge Cases:  
    - API throttling or authentication errors with Google Gemini  
    - Prompt expression or parsing issues  
    - Memory or model input mismatches  
  - Version Requirements: LangChain integration v1.8 or higher recommended

- **Google Gemini Chat Model**  
  - Type: Language Model Node (Google Gemini)  
  - Role: Provides fast, context-aware AI text generation using Google Gemini 2.0 Flash model  
  - Configuration:  
    - Model Name: `models/gemini-2.0-flash`  
    - Credentials: Google Palm API account (OAuth2-based)  
    - Options: Default  
  - Inputs: None (called by AI Agent)  
  - Outputs: AI Agent node (as language model input)  
  - Edge Cases:  
    - API authentication failures  
    - Rate limits or quota exhaustion  
    - Network timeouts  

- **Sticky Note1 (AI Processing Core)**  
  - Content: Highlights Google Gemini’s capabilities and role in the workflow  
  - Purpose: Documentation

---

#### 1.4 Response Output

**Overview:**  
Returns the AI-generated response to the original requester in a structured format, ensuring consistent formatting and frontend compatibility.

**Nodes Involved:**  
- Respond to Webhook  
- Sticky Note3 (Response Output)

**Node Details:**

- **Respond to Webhook**  
  - Type: HTTP Response Node  
  - Role: Sends back the AI assistant’s reply as the HTTP response to the original webhook call  
  - Configuration:  
    - Options: Default (returns data from AI Agent node)  
  - Inputs: AI Agent node output  
  - Outputs: None (terminal node)  
  - Edge Cases:  
    - If AI Agent output is malformed or missing, response could be empty or error  
    - HTTP connection issues to client

- **Sticky Note3 (Response Output)**  
  - Content: Describes response formatting and readiness for frontend use  
  - Purpose: Documentation

---

### 3. Summary Table

| Node Name            | Node Type                           | Functional Role               | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                   |
|----------------------|-----------------------------------|------------------------------|---------------------|----------------------|-----------------------------------------------------------------------------------------------|
| Webhook              | HTTP Webhook Trigger               | Entry point for user requests| —                   | AI Agent             | ## Webhook Entry Point\nReceives requests from multiple platforms; expects JSON with message and sessionId |
| Simple Memory        | LangChain Memory Buffer Window    | Maintains session memory     | —                   | AI Agent (memory)    | ## Session Memory\nConversation context stored per sessionId for natural follow-up            |
| AI Agent             | LangChain Agent                   | Core AI processing node      | Webhook, Simple Memory, Google Gemini Chat Model | Respond to Webhook | ## AI Processing Core\nPowered by Google Gemini 2.0 Flash for context-aware responses          |
| Google Gemini Chat Model | Language Model (Google Gemini)  | Provides AI text generation  | —                   | AI Agent             | ## AI Processing Core\nFast, context-aware language model                                    |
| Respond to Webhook   | HTTP Response Node                 | Sends AI response back       | AI Agent            | —                    | ## Response Output\nReturns structured, JSON formatted AI response                            |
| Sticky Note          | Sticky Note                       | Documentation                | —                   | —                    | ## Webhook Entry Point\nExplains expected payload and sources of webhook requests             |
| Sticky Note1         | Sticky Note                       | Documentation                | —                   | —                    | ## AI Processing Core\nDescribes Google Gemini capabilities                                  |
| Sticky Note2         | Sticky Note                       | Documentation                | —                   | —                    | ## Session Memory\nExplains memory buffer functionality                                      |
| Sticky Note3         | Sticky Note                       | Documentation                | —                   | —                    | ## Response Output\nDetails response formatting and use case                                |
| Sticky Note4         | Sticky Note                       | Documentation                | —                   | —                    | ## 🚀 AI Assistant Workflow\nSummary of integration options and usage instructions           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook Trigger  
   - Set HTTP Method to POST  
   - Set Webhook Path to a unique identifier (e.g., `45cec96e-962d-4e27-ab75-34d25b837032`)  
   - Enable Response Mode: Response Node (to delay response until data is ready)  
   - Save credentials if required (usually none for webhook)  

2. **Create Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Set Session Key Expression to: `={{ $json.body.sessionId }}`  
   - Set Session ID Type to: Custom Key  
   - No credentials required  

3. **Create Google Gemini Chat Model Node**  
   - Type: LangChain Google Gemini Chat Model  
   - Set Model Name to: `models/gemini-2.0-flash`  
   - Configure Google Palm API credentials (OAuth2 account with access to Gemini)  

4. **Create AI Agent Node**  
   - Type: LangChain Agent  
   - Set Prompt Type to: Define  
   - Enter prompt text:  
     ```
     You are a helpful AI assistant designed to provide accurate, concise, and friendly responses to user queries. Your goal is to understand the user's needs and provide relevant information, guidance, or solutions. Always maintain a professional yet approachable tone. If you're unsure about something, acknowledge it honestly and suggest alternative approaches when possible.
     ```  
   - Connect AI Agent's `ai_memory` input to Simple Memory node output  
   - Connect AI Agent's `ai_languageModel` input to Google Gemini Chat Model node output  

5. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Leave default options (returns incoming data as HTTP response)  

6. **Connect Nodes Sequentially**  
   - Webhook main output → AI Agent main input  
   - Simple Memory output → AI Agent ai_memory input  
   - Google Gemini Chat Model output → AI Agent ai_languageModel input  
   - AI Agent main output → Respond to Webhook main input  

7. **Add Sticky Notes for Documentation (Optional)**  
   - Add notes describing each block’s purpose, expected inputs, and outputs for maintainability and onboarding  

8. **Activate Workflow**  
   - Test by sending POST requests with JSON payloads:  
     ```json
     {
       "message": "Your question here",
       "sessionId": "unique-session-id"
     }
     ```  
   - Confirm AI responses are returned and context is maintained across messages with the same sessionId  

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow supports integration with website chat widgets, mobile apps, WhatsApp/Telegram bots, custom dashboards, and platforms | Workflow description and sticky notes                                |
| Google Gemini 2.0 Flash provides fast and context-aware language model capabilities ideal for conversational AI                   | Sticky Note1 content                                                  |
| Session memory uses a windowed buffer approach to maintain conversation context per sessionId                                     | Sticky Note2 content                                                  |
| The webhook expects POST requests with JSON containing `message` and `sessionId` fields                                            | Sticky Note (Webhook Entry Point)                                    |
| Workflow designed for professional yet friendly AI assistant responses, including honest fallback when unsure                    | AI Agent prompt text                                                  |
| For Google Gemini API, ensure valid Google Palm API credentials with proper OAuth2 setup                                           | Google Gemini Chat Model node credentials                            |
| Official n8n LangChain integration documentation can assist with advanced customization and troubleshooting                      | https://docs.n8n.io/integrations/agents/langchain-agent              |

---

**Disclaimer:**  
The text provided results exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.