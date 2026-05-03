Build an Interactive AI Agent with Chat Interface and Multiple Tools

https://n8nworkflows.xyz/workflows/build-an-interactive-ai-agent-with-chat-interface-and-multiple-tools-5819


# Build an Interactive AI Agent with Chat Interface and Multiple Tools

### 1. Workflow Overview

This workflow builds an interactive AI Agent accessible via a chat interface that can use multiple specialized tools to respond to user queries. It is designed to showcase the power of AI agents combined with functional automation, allowing users to interact conversationally and receive intelligent, context-aware responses leveraging various external APIs and internal computations.

The workflow is logically divided into the following blocks:

- **1.1 Chat Interface**: The public-facing chat window where users send messages and receive responses.
- **1.2 AI Agent Core (Brain)**: The central agent node that interprets user input, manages conversation context, decides which tool to use, and synthesizes responses.
- **1.3 AI Language Model(s)**: Large Language Model nodes providing natural language understanding and generation. Includes Gemini (Google Palm) by default, with an option for OpenAI.
- **1.4 Memory Management**: A memory buffer node that retains recent conversation history to provide context continuity.
- **1.5 Tools (Agent’s Toolbox)**: A set of specialized nodes acting as the agent’s capabilities or “superpowers,” including joke fetching, date calculations, Wikipedia search, password generation, loan payment calculation, and fetching n8n blog RSS feeds.
- **1.6 Informational Sticky Notes**: Documentation nodes embedded within the workflow for user guidance, branding, and feedback solicitation.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Interface

- **Overview:**  
This block provides the user interface for interaction with the AI agent. It hosts the chat window accessible via a public URL and handles incoming user messages.

- **Nodes Involved:**  
  - Example Chat Window

- **Node Details:**

  - **Example Chat Window**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Captures incoming chat messages and triggers the workflow; serves as the front-end chat interface.  
    - Configuration:  
      - `public` access enabled to allow anyone with the link to chat.  
      - Custom CSS defines a modern glassmorphism theme with branding colors, fonts, and UI styling for message bubbles and input areas.  
      - Shows no welcome screen on start.  
      - Uses “lastNode” response mode (responds with the last node’s output).  
      - Placeholder text: "Type your message here.."  
      - Initial greeting message: "Hi there! 👋"  
    - Inputs: Receives user chat messages via webhook.  
    - Outputs: Sends user messages to the AI Agent Core node.  
    - Edge Cases:  
      - Webhook ID must remain consistent and accessible.  
      - Public access could expose the interface to unsolicited inputs; rate limiting might be necessary depending on deployment.  
    - Sticky Notes: See Sticky Note1 for detailed usage instructions.

#### 2.2 AI Agent Core (Brain)

- **Overview:**  
Acts as the centralized decision-maker that understands the user’s intent, selects appropriate tools, manages the conversation flow, and generates responses.

- **Nodes Involved:**  
  - Your First AI Agent

- **Node Details:**

  - **Your First AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Core AI agent logic node integrating memory, language models, and tools.  
    - Configuration:  
      - Contains a comprehensive system message defining the agent’s personality, instructions, and tool knowledge.  
      - The system message sets the agent as a demo AI assistant named “n8n Demo AI Agent” created by Lucas Peyrin.  
      - Provides detailed instructions for tool usage and behavioral guidelines, including available tools and fallback responses.  
      - Output format mandates friendly, conversational responses with proactive suggestions.  
      - Enables contextual memory usage via the connected memory node.  
    - Inputs: Receives messages from the chat window.  
    - Outputs: Sends responses back to the chat window.  
    - Connects to multiple tool nodes via its ai_tool input to invoke specific capabilities.  
    - Connects to the language model node via ai_languageModel input for generating natural language.  
    - Connects to the memory node via ai_memory input to maintain conversation context.  
    - Edge Cases:  
      - Misconfiguration or missing tools can cause failed tool invocations.  
      - Overly complex or ambiguous user inputs may lead to incorrect tool selection.  
      - System message must be properly maintained to ensure agent coherence.  
    - Sticky Notes: See Sticky Note2 for role explanation.

#### 2.3 AI Language Model(s)

- **Overview:**  
Provides the natural language understanding and generation backbone for the AI agent. Gemini is used by default, with OpenAI GPT-4 available but disabled.

- **Nodes Involved:**  
  - Gemini  
  - OpenAI (disabled)

- **Node Details:**

  - **Gemini**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
    - Role: Primary LLM model generating responses and understanding.  
    - Configuration:  
      - Model: “models/gemini-2.5-flash”  
      - Temperature set to 0 for deterministic output.  
      - Requires Google Palm API credentials (configured via “IA2S” credential).  
    - Inputs: Connected to AI Agent Core’s ai_languageModel input.  
    - Outputs: Text completions / chat responses to the agent.  
    - Edge Cases:  
      - API key expiration or invalid credentials cause authentication errors.  
      - Rate limiting or API downtime can cause timeouts or failures.  
    - Sticky Notes: See Sticky Note3 for configuration guidance.

  - **OpenAI** (disabled)  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Alternative LLM option (GPT-4.1-mini).  
    - Configuration:  
      - Disabled by default; requires manual activation and credential setup.  
      - Temperature set to 0 for deterministic output.  
    - Edge Cases: Same as Gemini with OpenAI API specifics.  
    - Inputs/Outputs: Similar to Gemini but disconnected due to disabled status.

#### 2.4 Memory Management

- **Overview:**  
Handles short-term memory by buffering recent conversation messages, allowing the agent to maintain context across exchanges.

- **Nodes Involved:**  
  - Simple Memory

- **Node Details:**

  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains a sliding window of recent messages to provide context.  
    - Configuration:  
      - Context window length set to 30 messages, balancing memory depth and performance.  
    - Inputs: Connected from agent node’s ai_memory output.  
    - Outputs: Provides context back to the agent node.  
    - Edge Cases:  
      - Excessive context window length can slow down processing or exceed token limits of LLM.  
      - Insufficient context length may cause loss of conversation continuity.  
    - Sticky Notes: See Sticky Note4 for explanation.

#### 2.5 Tools (Agent’s Toolbox)

- **Overview:**  
A suite of specialized nodes providing distinct functionalities that the AI agent can invoke as “tools” to fulfill user requests.

- **Nodes Involved:**  
  - get_a_joke  
  - days_from_now  
  - wikipedia  
  - create_password  
  - calculate_loan_payment  
  - n8n_blog_rss_feed

- **Node Details:**

  - **get_a_joke**  
    - Type: `n8n-nodes-base.httpRequestTool`  
    - Role: Fetches a clean, safe joke from an external Joke API.  
    - Configuration:  
      - Uses https://v2.jokeapi.dev/joke/Any with blacklist for NSFW and sensitive content.  
      - Requests a single-part joke (field: "joke").  
      - Optimizes response to include only the selected field.  
    - Inputs: Invoked by agent via ai_tool.  
    - Outputs: Returns joke text to the agent.  
    - Edge Cases:  
      - API downtime or network errors.  
      - Unexpected response format.  

  - **days_from_now**  
    - Type: `n8n-nodes-base.dateTimeTool`  
    - Role: Calculates the difference in days between current date and a specified future date.  
    - Configuration:  
      - Start date fixed as current time (`$now`).  
      - End date is user-provided via AI override prompt to specify.  
      - Outputs difference as `now_day_difference`.  
    - Edge Cases:  
      - Invalid or malformed date inputs.  
      - Timezone discrepancies.  

  - **wikipedia**  
    - Type: `@n8n/n8n-nodes-langchain.toolWikipedia`  
    - Role: Searches and summarizes Wikipedia articles for a topic.  
    - Configuration: Default settings with no parameters.  
    - Edge Cases:  
      - Nonexistent or ambiguous topics.  
      - API availability issues.  

  - **create_password**  
    - Type: `n8n-nodes-base.cryptoTool`  
    - Role: Generates a secure random password.  
    - Configuration:  
      - Generates base64-encoded string.  
      - Length configurable via AI override prompt, recommended between 8 to 16 characters.  
      - Outputs password in `password` field.  
    - Edge Cases:  
      - Length set too short or too long may reduce security or usability.  

  - **calculate_loan_payment**  
    - Type: `@n8n/n8n-nodes-langchain.toolCode`  
    - Role: Calculates fixed monthly loan payments using amortization formula.  
    - Configuration:  
      - JavaScript code expects loan amount, annual interest rate (%), and term in years.  
      - Validates inputs to be positive numbers; errors returned otherwise.  
      - Outputs JSON string with monthly payment, total paid, and total interest.  
    - Edge Cases:  
      - Invalid or missing input parameters.  
      - Division by zero or invalid math formula application.  

  - **n8n_blog_rss_feed**  
    - Type: `n8n-nodes-base.rssFeedReadTool`  
    - Role: Fetches latest posts from the official n8n blog RSS feed.  
    - Configuration:  
      - RSS URL: https://n8n.io/blog/rss  
    - Edge Cases:  
      - RSS feed unavailable or changed format.  
      - Network errors.  

- All tool nodes connect their output to the agent node’s ai_tool input, allowing the agent to dynamically invoke them as needed.

- Sticky Notes: See Sticky Note5 for conceptual explanation.

#### 2.6 Informational Sticky Notes

- **Overview:**  
These nodes provide embedded documentation, instructions, and branding messages to aid users and developers.

- **Nodes Involved:**  
  - Sticky Note1 to Sticky Note10

- **Node Details:**  
Each sticky note contains detailed explanations covering:

  - Chat interface use and customization (Sticky Note1)  
  - AI agent’s role and behavior (Sticky Note2)  
  - Language model configuration instructions (Sticky Note3)  
  - Memory node purpose (Sticky Note4)  
  - Tools explanation and extensibility (Sticky Note5)  
  - Feedback solicitation and coaching/consulting offers with links (Sticky Note10)

- Positioning of sticky notes is near their related functional nodes for easy reference.

---

### 3. Summary Table

| Node Name            | Node Type                                      | Functional Role               | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                                   |
|----------------------|------------------------------------------------|------------------------------|-----------------------|-----------------------|------------------------------------------------------------------------------------------------------------------------------|
| Example Chat Window   | @n8n/n8n-nodes-langchain.chatTrigger          | Public chat interface          | Webhook (external)     | Your First AI Agent    | See Sticky Note1: Chat interface instructions and customization details.                                                     |
| Your First AI Agent   | @n8n/n8n-nodes-langchain.agent                 | AI agent core brain            | Example Chat Window, Gemini, Simple Memory, Tools  | Example Chat Window      | See Sticky Note2: Agent role, decision-making, and personality explanation.                                                  |
| Gemini               | @n8n/n8n-nodes-langchain.lmChatGoogleGemini    | Primary Language Model         | Your First AI Agent    | Your First AI Agent    | See Sticky Note3: LLM configuration and credential setup instructions.                                                      |
| OpenAI (disabled)     | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Alternative LLM (disabled)     | None                  | None                  | See Sticky Note3: Alternative LLM option, disabled by default.                                                              |
| Simple Memory         | @n8n/n8n-nodes-langchain.memoryBufferWindow    | Conversation context memory    | Your First AI Agent    | Your First AI Agent    | See Sticky Note4: Memory role and context window explanation.                                                                |
| get_a_joke            | n8n-nodes-base.httpRequestTool                  | Tool: Fetches jokes            | Your First AI Agent    | Your First AI Agent    | See Sticky Note5: Tools explanation and extensibility.                                                                       |
| days_from_now         | n8n-nodes-base.dateTimeTool                      | Tool: Date difference calculator | Your First AI Agent    | Your First AI Agent    | See Sticky Note5                                                                                                              |
| wikipedia             | @n8n/n8n-nodes-langchain.toolWikipedia          | Tool: Wikipedia search         | Your First AI Agent    | Your First AI Agent    | See Sticky Note5                                                                                                              |
| create_password       | n8n-nodes-base.cryptoTool                        | Tool: Password generator       | Your First AI Agent    | Your First AI Agent    | See Sticky Note5                                                                                                              |
| calculate_loan_payment| @n8n/n8n-nodes-langchain.toolCode                | Tool: Loan payment calculator  | Your First AI Agent    | Your First AI Agent    | See Sticky Note5                                                                                                              |
| n8n_blog_rss_feed     | n8n-nodes-base.rssFeedReadTool                   | Tool: Fetches n8n blog posts   | Your First AI Agent    | Your First AI Agent    | See Sticky Note5                                                                                                              |
| Sticky Note1          | n8n-nodes-base.stickyNote                        | Documentation: Chat Interface  | None                  | None                  | See content about chat window usage and testing.                                                                              |
| Sticky Note2          | n8n-nodes-base.stickyNote                        | Documentation: AI Agent Core   | None                  | None                  | See content describing agent’s brain and tool selection.                                                                     |
| Sticky Note3          | n8n-nodes-base.stickyNote                        | Documentation: LLMs            | None                  | None                  | See content about choosing and configuring LLM nodes.                                                                        |
| Sticky Note4          | n8n-nodes-base.stickyNote                        | Documentation: Memory          | None                  | None                  | Explains the importance and configuration of the memory node.                                                                |
| Sticky Note5          | n8n-nodes-base.stickyNote                        | Documentation: Tools           | None                  | None                  | Explains the agent's toolbox and how to add or modify tools.                                                                  |
| Sticky Note10         | n8n-nodes-base.stickyNote                        | Documentation: Feedback & Coaching | None                  | None                  | Contains feedback form and consulting service links: https://api.ia2s.app/form/templates/feedback?template=First%20AI%20Agent and others.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Interface Node**  
   - Add a `ChatTrigger` node (`@n8n/n8n-nodes-langchain.chatTrigger`) named "Example Chat Window".  
   - Set `public` to true for open access.  
   - Customize title: "Your first AI Agent 🚀", subtitle: "This is for demo purposes. Try me out !".  
   - Paste the provided custom CSS for glassmorphism styling in the Custom CSS field.  
   - Set `responseMode` to "lastNode".  
   - Set input placeholder: "Type your message here..".  
   - Disable welcome screen and set initial message: "Hi there! 👋".  
   - Save webhook URL for testing.

2. **Set Up the AI Agent Core Node**  
   - Add an `Agent` node (`@n8n/n8n-nodes-langchain.agent`) named "Your First AI Agent".  
   - Paste the provided detailed system message defining personality, instructions, tools, and output format.  
   - Connect the "Example Chat Window" node’s main output to this node’s main input.  
   - This node will manage tool selection, memory, and interaction logic.

3. **Add AI Language Model Node**  
   - Add a `Google Gemini` node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) named "Gemini".  
   - Set model name to "models/gemini-2.5-flash".  
   - Set temperature to 0 for deterministic responses.  
   - Attach your Google Palm API credential under `googlePalmApi`.  
   - Connect this node’s output to "Your First AI Agent" node’s `ai_languageModel` input.  
   - (Optionally) Add an `OpenAI` node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`), disabled by default, configured with your OpenAI API credentials.

4. **Configure Memory Node**  
   - Add a `Memory Buffer Window` node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`) named "Simple Memory".  
   - Set `contextWindowLength` to 30 messages.  
   - Connect this node’s output to the `ai_memory` input of "Your First AI Agent".  
   - Connect the agent node to this node to enable memory context management.

5. **Add Tool Nodes**  
   - **get_a_joke**: Add an `HTTP Request Tool` node. Set URL to "https://v2.jokeapi.dev/joke/Any?blacklistFlags=nsfw,religious,political,racist,sexist,explicit&type=single", request field "joke", optimize response.  
   - **days_from_now**: Add a `DateTime Tool` node. Operation: "getTimeBetweenDates". Start date: current time `$now`. End date: AI prompt input. Output field name: `now_day_difference`.  
   - **wikipedia**: Add a `Wikipedia Tool` node with default settings.  
   - **create_password**: Add a `Crypto Tool` node. Action: "generate". Encoding: "base64". String length: AI prompt input (8-16 recommended). Output property: `password`.  
   - **calculate_loan_payment**: Add a `Code Tool` node with the provided JavaScript loan amortization code. Configure input schema expecting loan_amount, annual_rate, term_years.  
   - **n8n_blog_rss_feed**: Add an `RSS Feed Read Tool` node. URL: "https://n8n.io/blog/rss".  

   - Connect all these tool nodes’ outputs to the `ai_tool` input of the "Your First AI Agent" node.

6. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add `Sticky Note` nodes near each major block with the provided contents for instructions, branding, and links.

7. **Final Connections and Testing**  
   - Connect "Example Chat Window" main output to "Your First AI Agent" main input.  
   - Connect "Your First AI Agent" main output back to "Example Chat Window" for responses.  
   - Ensure all API credentials are correctly configured (Google Palm API for Gemini, any others as needed).  
   - Activate the workflow.  
   - Use the webhook URL from "Example Chat Window" to open the chat interface in a browser.  
   - Test by sending messages and verifying the agent responds appropriately using tools.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates an AI agent with conversational UI and multiple callable tools, built by Lucas Peyrin. It is designed to be a reusable template for building AI-powered chatbots in n8n.                                                                                                                                                                    | Branding and project credits.                                                                                                                                                   |
| Feedback link for users: "Was this helpful? Let me know!" with form to submit feedback on the template.                                                                                                                                                                                                                                                                 | https://api.ia2s.app/form/templates/feedback?template=First%20AI%20Agent                                                                                                         |
| Coaching sessions are offered to advance n8n skills with personalized help.                                                                                                                                                                                                                                                                                             | https://api.ia2s.app/form/templates/coaching?template=First%20AI%20Agent                                                                                                         |
| Consulting services for complex project development are available.                                                                                                                                                                                                                                                                                                      | https://api.ia2s.app/form/templates/consulting?template=First%20AI%20Agent                                                                                                       |
| Lucas Peyrin's n8n Templates collection link for additional learning and examples.                                                                                                                                                                                                                                                                                      | https://n8n.io/creators/lucaspeyrin                                                                                                                                             |
| The workflow uses a glassmorphism CSS theme for the chat interface, creating a modern and visually appealing user experience.                                                                                                                                                                                                                                         | Custom CSS embedded in the "Example Chat Window" node.                                                                                                                         |
| Important: For production use, consider securing the chat webhook, handling API quota limits, and monitoring for errors or misuse.                                                                                                                                                                                                                                     | Operational best practices.                                                                                                                                                      |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.