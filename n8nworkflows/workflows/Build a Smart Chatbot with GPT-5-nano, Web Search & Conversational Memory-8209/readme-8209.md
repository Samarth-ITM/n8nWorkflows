Build a Smart Chatbot with GPT-5-nano, Web Search & Conversational Memory

https://n8nworkflows.xyz/workflows/build-a-smart-chatbot-with-gpt-5-nano--web-search---conversational-memory-8209


# Build a Smart Chatbot with GPT-5-nano, Web Search & Conversational Memory

### 1. Workflow Overview

This workflow implements a smart conversational chatbot named "Sofia" using the GPT-5-nano language model, integrated web search capabilities, and conversational memory. It is designed to receive chat messages from users, process them intelligently with AI, enhance responses with real-time web search data, and maintain conversational context across sessions. The user interface is clean and minimalistic, styled with custom CSS for a modern chat experience.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Session Memory:** Captures incoming user chat messages and manages session-based conversation memory.

- **1.2 AI Processing (Agent):** Processes the input using an AI agent that combines GPT-5-nano, web search, and memory buffers to generate a contextually rich response.

- **1.3 Response Delivery:** Sends the generated AI response back to the user and ensures conversation continuity.

Additional elements include sticky notes for documentation and licensing, and a comprehensive welcome screen with customized UI.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Session Memory

**Overview:**  
This block triggers the workflow upon receiving a chat message from a user, manages session state, and stores user conversation history in memory buffers to maintain context across messages and sessions.

**Nodes Involved:**  
- When chat message received  
- Simple Memory

**Node Details:**

- **When chat message received**  
  - *Type:* `@n8n/n8n-nodes-langchain.chatTrigger`  
  - *Role:* Webhook trigger node for incoming chat messages. It initiates the workflow when the user sends a message.  
  - *Configuration:*  
    - Publicly accessible webhook with custom title "Welcome to Sofia" and subtitle "Start a chat. We're here to help you 24/7."  
    - Custom CSS styles for a clean, modern chat UI with a soft color palette and responsive layout.  
    - Initial greeting message configured: "Hi there! 👋 My name is Sofia. How can I assist you today?"  
    - Session memory enabled with up to 5 sessions loaded and persisted.  
    - Input placeholder text: "Type your question.."  
  - *Input:* User chat messages via webhook  
  - *Output:* Message data and session context passed downstream  
  - *Edge cases:*  
    - Network or webhook connectivity issues may prevent triggering.  
    - Session memory overflow if session limit exceeded (default 5 sessions).  
    - Expression or CSS parsing errors in custom UI styling.  

- **Simple Memory**  
  - *Type:* `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - *Role:* Stores raw user input and conversation snippets to maintain session context.  
  - *Configuration:* Default buffer window length (not explicitly set, defaults apply).  
  - *Input:* Connected from "When chat message received" node’s AI memory output.  
  - *Output:* Memory context used by the AI Agent for generating responses.  
  - *Edge cases:*  
    - Memory buffer may lose context if conversation is too long or if not properly persisted.

---

#### 2.2 AI Processing (Agent)

**Overview:**  
This block forms the core intelligence of the chatbot. It uses an AI agent node that combines OpenAI's GPT-5-nano model, web search via Bing, and conversational memory buffers to generate accurate, context-aware, and up-to-date responses.

**Nodes Involved:**  
- AI Agent  
- GPT  
- IA Memory  
- Search

**Node Details:**

- **AI Agent**  
  - *Type:* `@n8n/n8n-nodes-langchain.agent`  
  - *Role:* Central orchestrator AI node that integrates language modeling, memory, and tool-based search to produce responses.  
  - *Configuration:*  
    - System message set to "You are a helpful assistant" guiding response style.  
    - Accepts references to language model, memory, and tools as inputs.  
  - *Input:*  
    - Receives memory context from "IA Memory" node.  
    - Receives language model from "GPT" node.  
    - Receives tool (search) from "Search" node.  
  - *Output:* AI-generated message sent to "Respond to Chat".  
  - *Edge cases:*  
    - Authentication errors with OpenAI API or Bing search.  
    - Timeout or rate limits on API calls.  
    - Misconfiguration of system message or missing memory inputs leading to incoherent responses.  

- **GPT**  
  - *Type:* `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - *Role:* Provides AI language model responses using OpenAI GPT-5-nano model.  
  - *Configuration:*  
    - Model: GPT-5-nano selected for lightweight and fast generation.  
    - Temperature set to 0.7 for creative yet coherent responses.  
    - Credentials: Configured with OpenAI API key.  
  - *Input:* Receives prompts from "AI Agent" node.  
  - *Output:* Text completions passed back to "AI Agent".  
  - *Edge cases:*  
    - API key invalidation or quota exhaustion.  
    - Model unavailability or version mismatch.  

- **IA Memory**  
  - *Type:* `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - *Role:* Maintains a sliding window of conversation context (last 10 messages) used by the AI agent.  
  - *Configuration:* Context window length set to 10 messages for balanced context size.  
  - *Input:* Conversation data from "AI Agent".  
  - *Output:* Provides memory context back to "AI Agent".  
  - *Edge cases:*  
    - Context length may truncate important conversation history.  
    - Memory synchronization issues with concurrent sessions.  

- **Search**  
  - *Type:* `n8n-nodes-base.httpRequestTool`  
  - *Role:* Provides real-time web search results using Bing search engine to enrich AI responses with current information.  
  - *Configuration:*  
    - URL is dynamically set via AI override expression (placeholder for actual search query).  
    - Tool description references Bing search example: `https://www.bing.com/search?q=example`  
  - *Input:* Search query from "AI Agent".  
  - *Output:* Search results fed back to "AI Agent" as a tool result.  
  - *Edge cases:*  
    - HTTP errors or Bing API limitations.  
    - Malformed search queries or empty results.  

---

#### 2.3 Response Delivery

**Overview:**  
This block sends the AI-generated response back to the user through the chat interface and ensures the conversation is updated accordingly.

**Nodes Involved:**  
- Respond to Chat

**Node Details:**

- **Respond to Chat**  
  - *Type:* `@n8n/n8n-nodes-langchain.chat`  
  - *Role:* Outputs the AI agent’s response into the chat UI for the user to see.  
  - *Configuration:*  
    - Message content dynamically set to the output from the AI Agent node (`={{ $json.output }}`).  
    - No additional options configured.  
  - *Input:* Receives AI text output from "AI Agent".  
  - *Output:* Sends message to chat interface and completes the workflow cycle.  
  - *Edge cases:*  
    - Failures if output is empty or malformed.  
    - Network or UI communication errors.  

---

### 3. Summary Table

| Node Name               | Node Type                                    | Functional Role                      | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                      |
|-------------------------|----------------------------------------------|------------------------------------|-----------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger        | Input Reception & Session Memory   | -                           | AI Agent, Simple Memory     | ## Chat When a chat message is received: Triggers the flow when a user sends a message. It also manages appearance and initial setup. Also has session memory. |
| Simple Memory           | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores raw user session memory     | When chat message received  | When chat message received  |                                                                                                |
| AI Agent                | @n8n/n8n-nodes-langchain.agent               | AI Processing Core                 | When chat message received, GPT, IA Memory, Search | Respond to Chat            | ## Agent AI Agent: The brain of the chatbot. Uses memory, GPT, and Bing search to generate intelligent responses. |
| GPT                     | @n8n/n8n-nodes-langchain.lmChatOpenAi        | Language Model                    | AI Agent                    | AI Agent                   |                                                                                                |
| IA Memory               | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversational Memory Buffer      | AI Agent                    | AI Agent                   |                                                                                                |
| Search                  | n8n-nodes-base.httpRequestTool                | Web Search Tool                   | AI Agent                    | AI Agent                   |                                                                                                |
| Respond to Chat         | @n8n/n8n-nodes-langchain.chat                 | Response Delivery                 | AI Agent                    | -                          | ## Response Respond to Chat: Sends the AI-generated message back to the user and stores conversation. |
| Sticky Note             | n8n-nodes-base.stickyNote                      | Documentation                    | -                           | -                          | ## Chat When a chat message is received: Triggers the flow when a user sends a message. It also manages the appearance and initial setup. It also has session memory, so it remembers up to 5 sessions (this can be increased). |
| Sticky Note1            | n8n-nodes-base.stickyNote                      | Documentation                    | -                           | -                          | ## Agent AI Agent: The brain of the chatbot. It uses memory, the GPT model, and Bing's search engine to generate an intelligent and configurable response. |
| Sticky Note2            | n8n-nodes-base.stickyNote                      | Documentation                    | -                           | -                          | ## Response Respond to Chat: Sends the AI-generated message back to the user in the chat and stores the conversation in memory. |
| Sticky Note3            | n8n-nodes-base.stickyNote                      | Documentation                    | -                           | -                          | ## BY OXSR By OXSR, visit my profile for professional and business workflows https://n8n.io/creators/oxsr11/ Git: https://github.com/OXSR Mail: oriolrotllant3@gmail.com |
| Sticky Note4            | n8n-nodes-base.stickyNote                      | Documentation                    | -                           | -                          | ## Welcome Este flujo de trabajo está diseñado para crear un chatbot inteligente y funcional. Al recibir un mensaje, el chatbot utiliza una potente IA con memoria y la capacidad de buscar en la web para generar respuestas precisas y conversacionales. La respuesta se envía de vuelta al usuario con un diseño limpio y minimalista, perfecto para una experiencia de usuario moderna y fluida. |
| Sticky Note5            | n8n-nodes-base.stickyNote                      | Documentation                    | -                           | -                          | ## LICENSE This project is distributed under an open source license. You are free to use, modify, and distribute this work. Key Terms: Use: You may use this project for personal, educational, or commercial purposes. Modification: You are permitted to modify the code or content as you wish. Distribution: You may share copies of the project, both in its original and modified versions. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "When chat message received" node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhook to be public.  
   - Title: "Welcome to Sofia"  
   - Subtitle: "Start a chat. We're here to help you 24/7."  
   - Paste the provided custom CSS into the options for styling the chat UI.  
   - Set initial welcome message: "Hi there! 👋 My name is Sofia. How can I assist you today?"  
   - Enable session memory with a limit of 5 sessions (adjustable).  
   - Set input placeholder: "Type your question.."  
   - Enable loading previous session from memory.  

2. **Add "Simple Memory" node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Use default parameters (buffer window length default).  
   - Connect the AI memory output of "When chat message received" to this node’s input.  
   - Connect this node’s output back to the AI memory input of "When chat message received" to maintain session memory.  

3. **Add "AI Agent" node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set the system message to: "You are a helpful assistant".  
   - The agent requires inputs: language model, memory, and tool for search.  

4. **Add "GPT" node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: Select "gpt-5-nano".  
   - Temperature: 0.7  
   - Credentials: Configure with your OpenAI API key.  
   - Connect the output of this node as the language model input to the "AI Agent" node.  

5. **Add "IA Memory" node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set context window length to 10 messages.  
   - Connect this node’s output as the memory input to the "AI Agent" node.  
   - Connect the "AI Agent" node’s conversation output to this node for updating the memory buffer.  

6. **Add "Search" node**  
   - Type: `n8n-nodes-base.httpRequestTool`  
   - URL: Use an expression placeholder that will be dynamically set by the AI agent, e.g., `={{ $fromAI('URL', '', 'string') }}`  
   - Set tool description to indicate it uses Bing search, e.g., "Search using Bing example: https://www.bing.com/search?q=example"  
   - Connect the output of this node as the tool input to the "AI Agent" node.  

7. **Connect "When chat message received" node main output to "AI Agent" node main input**  
   - This establishes the message flow from input to AI processing.  

8. **Add "Respond to Chat" node**  
   - Type: `@n8n/n8n-nodes-langchain.chat`  
   - Configure message to be dynamic: `={{ $json.output }}` (to send AI result).  
   - Connect the main output of "AI Agent" node to this node’s main input.  

9. **Final Connections:**  
   - Connect AI Agent outputs properly to memory and response nodes as described.  
   - Ensure all credentials for OpenAI and any HTTP requests are configured correctly.  

10. **Optional: Add sticky note nodes for documentation**  
    - Add sticky notes with the content from the original workflow for clarity and project info.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is designed to create a smart and functional chatbot using powerful AI with memory and web search capabilities, delivering precise, conversational responses in a clean and minimalistic design.           | Workflow purpose and UI design overview (Sticky Note4)                                             |
| AI Agent acts as the central brain, intelligently combining GPT-5-nano with Bing search and memory buffers to generate rich responses.                                                                               | Explanation of AI Agent node (Sticky Note1)                                                       |
| The project is open source with permissions to use, modify, and distribute freely for personal, educational, or commercial purposes.                                                                                  | License details (Sticky Note5)                                                                     |
| Developed by OXSR, who offers professional workflows and can be contacted via n8n profile, GitHub, or email.                                                                                                           | Creator info: https://n8n.io/creators/oxsr11/, https://github.com/OXSR, oriolrotllant3@gmail.com    |
| For examples of Bing search usage in HTTP requests, see: https://www.bing.com/search?q=example                                                                                                                        | Search node tool description                                                                        |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. It fully complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.