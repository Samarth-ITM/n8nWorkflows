Email Drafting & News Research Assistant with OpenAI, Gmail, Tavily & Perplexity

https://n8nworkflows.xyz/workflows/email-drafting---news-research-assistant-with-openai--gmail--tavily---perplexity-7679


# Email Drafting & News Research Assistant with OpenAI, Gmail, Tavily & Perplexity

### 1. Workflow Overview

This workflow, titled **"Email Drafting & News Research Assistant with OpenAI, Gmail, Tavily & Perplexity"**, is designed to streamline research and email composition tasks by combining advanced AI reasoning, up-to-date news search, and email automation within a single chat-driven interface.

**Target Use Cases:**
- Users initiate a chat prompt requesting news research and/or email drafting.
- The system performs real-time news searches using Tavily and Perplexity tools.
- It drafts emails based on AI reasoning and memory of the conversation.
- Emails are sent via Gmail integration.
- The workflow is suitable for professionals needing quick synthesis of news insights into email communication without switching between apps.

**Logical Blocks:**

- **1.1 Input Reception**: Captures user prompts via chat.
- **1.2 AI Reasoning and Memory**: Processes and plans responses using OpenAI and stores conversational context.
- **1.3 MCP Client Orchestration**: Connects the AI agent to Modular Cognitive Process (MCP) servers for news research and email drafting.
- **1.4 MCP Servers (Email & News Research)**: Backend services exposing tools for Gmail email sending and news search (Tavily + Perplexity).
- **1.5 External API Calls**: Nodes that interact with Gmail, Tavily, and Perplexity APIs.
- **1.6 Informational Sticky Notes**: Documentation nodes embedded for user guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview**: Listens for incoming chat messages that trigger the entire workflow.
- **Nodes Involved**:  
  - *When chat message received*

- **Node Details**:

  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry point for chat-based user input.  
    - Configuration: Default trigger settings to capture all incoming chat messages.  
    - Inputs: None  
    - Outputs: Connected to AI Agent node  
    - Edge Cases: Incoming message format issues or connectivity problems can prevent triggering.

---

#### 1.2 AI Reasoning and Memory

- **Overview**: This block uses OpenAI’s GPT-4 model to understand and plan responses, while maintaining short-term conversational memory to ensure context-aware multi-turn interactions.
- **Nodes Involved**:  
  - *AI Agent*  
  - *OpenAI Chat Model*  
  - *Simple Memory*

- **Node Details**:

  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Central orchestrator interpreting chat input and deciding which tools to invoke.  
    - Configuration: System message sets the assistant as a “helpful email assistant,” instructing it to use attached Email MCP Tool for email-related tasks.  
    - Inputs: From chat trigger  
    - Outputs: Connects to MCP Clients and OpenAI Chat Model  
    - Edge Cases: Misinterpretation of system prompt or tool communication failures.

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides reasoning and planning capabilities using GPT-4.1-mini model variant.  
    - Configuration: Model set to "gpt-4.1-mini" for balanced performance.  
    - Inputs: From AI Agent  
    - Outputs: To AI Agent for further processing  
    - Edge Cases: API rate limits, authentication errors, or model timeout.

  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains a windowed buffer of recent conversation messages to keep context.  
    - Configuration: Default buffer window size; stores recent exchanges.  
    - Inputs: From AI Agent (memory integration)  
    - Outputs: Back to AI Agent  
    - Edge Cases: Memory overflow or stale context if conversation is too long.

---

#### 1.3 MCP Client Orchestration

- **Overview**: These nodes represent the AI Agent’s interface to external MCP servers, allowing the agent to delegate research and email composition tasks.
- **Nodes Involved**:  
  - *Email MCP Client*  
  - *News MCP Client*

- **Node Details**:

  - **Email MCP Client**  
    - Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
    - Role: Connects to the Email MCP Server to handle email-related tasks like composing and sending emails.  
    - Configuration: Endpoint URL placeholder (to be replaced); transport set to HTTP streamable for real-time communication.  
    - Inputs: From AI Agent  
    - Outputs: To AI Agent  
    - Edge Cases: Server unreachable, incorrect endpoint, or transport protocol mismatch.

  - **News MCP Client**  
    - Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
    - Role: Connects to the News MCP Server to perform news searches and retrieve up-to-date information.  
    - Configuration: Endpoint URL placeholder; uses HTTP streamable transport.  
    - Inputs: From AI Agent  
    - Outputs: To AI Agent  
    - Edge Cases: Same as Email MCP Client.

---

#### 1.4 MCP Servers (Email & News Research)

- **Overview**: Backend MCP servers expose tools allowing the AI Agent to perform specialized tasks: email sending via Gmail and news research via Tavily and Perplexity.
- **Nodes Involved**:  
  - *Email MCP Server*  
  - *News MCP Server*

- **Node Details**:

  - **Email MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Exposes Gmail Tool nodes for sending emails based on AI instructions.  
    - Configuration: Path placeholder (to be replaced with public endpoint path).  
    - Inputs: From Email MCP Client tools  
    - Outputs: To Gmail Tool nodes  
    - Edge Cases: Endpoint misconfiguration, credential issues, or Gmail API errors.

  - **News MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Exposes Tavily and Perplexity tools to perform news search and answer queries.  
    - Configuration: Path placeholder to be replaced with actual server endpoint.  
    - Inputs: From News MCP Client tools  
    - Outputs: To Tavily and Perplexity nodes  
    - Edge Cases: Connectivity problems, API quota exhaustion, or query formatting errors.

---

#### 1.5 External API Calls

- **Overview**: These nodes interact directly with external services to perform core functionalities like sending emails or querying news.
- **Nodes Involved**:  
  - *Send a message in Gmail*  
  - *Send a message in Gmail1*  
  - *Send a message in Gmail2*  
  - *Search in Tavily*  
  - *Search in Tavily1*  
  - *Message a model in Perplexity*

- **Node Details**:

  - **Send a message in Gmail / Gmail1 / Gmail2**  
    - Type: `n8n-nodes-base.gmailTool`  
    - Role: Send emails via Gmail API with dynamic fields for recipient, subject, and message.  
    - Configuration: Placeholders for "To," "Subject," and "Message" which are filled dynamically by AI.  
    - Inputs: From Email MCP Server (triggered by AI commands)  
    - Outputs: None (final action node)  
    - Credential Requirements: Valid Gmail OAuth2 credentials connected in n8n.  
    - Edge Cases: Authentication failures, quota limits, invalid email addresses, or message formatting issues.

  - **Search in Tavily / Tavily1**  
    - Type: `@tavily/n8n-nodes-tavily.tavilyTool`  
    - Role: Perform search queries on Tavily platform to retrieve latest news or information.  
    - Configuration: Queries are dynamically injected from AI outputs (using `$fromAI` expressions).  
    - Inputs: From News MCP Server  
    - Outputs: Back to News MCP Server  
    - Credential Requirements: Tavily API credentials configured.  
    - Edge Cases: API key invalid, no results found, or malformed queries.

  - **Message a model in Perplexity**  
    - Type: `n8n-nodes-base.perplexityTool`  
    - Role: Sends queries to Perplexity AI for retrieval-augmented generation or simplified answers.  
    - Configuration: Message content and simplify flag dynamically injected via AI expressions.  
    - Inputs: From News MCP Server  
    - Outputs: Back to News MCP Server  
    - Credential Requirements: Perplexity API credentials connected.  
    - Edge Cases: Rate limiting, connection timeouts, or unexpected response format.

---

#### 1.6 Informational Sticky Notes

- **Overview**: Provide embedded documentation, usage instructions, and references for users editing or running the workflow.
- **Nodes Involved**:  
  - *Sticky Note*  
  - *Sticky Note1*  
  - *Sticky Note2*  
  - *Sticky Note3*

- **Node Details**:

  - **Sticky Note** (Large, detailed)  
    - Purpose: Explains the overall workflow, including components, use cases, setup instructions, and best practices.  
    - Location: Top-left corner for easy visibility.

  - **Sticky Note1**  
    - Content: Labels the cluster containing the AI Agent and MCP Clients.

  - **Sticky Note2**  
    - Content: Labels the Email MCP Server nodes.

  - **Sticky Note3**  
    - Content: Labels the News Research MCP Server nodes.

---

### 3. Summary Table

| Node Name                 | Node Type                                  | Functional Role                  | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                                  |
|---------------------------|--------------------------------------------|--------------------------------|--------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received | `@n8n/n8n-nodes-langchain.chatTrigger`    | Input reception from chat      | None                     | AI Agent                  |                                                                                                              |
| AI Agent                  | `@n8n/n8n-nodes-langchain.agent`           | Orchestrates tools via MCP     | When chat message received| OpenAI Chat Model, MCP Clients | Sticky Note1: "Agent & MCP Client"                                                                            |
| OpenAI Chat Model          | `@n8n/n8n-nodes-langchain.lmChatOpenAi`   | AI reasoning and planning      | AI Agent                 | AI Agent                  |                                                                                                              |
| Simple Memory              | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Maintains conversation context | AI Agent (memory)         | AI Agent                  |                                                                                                              |
| Email MCP Client           | `@n8n/n8n-nodes-langchain.mcpClientTool`  | Connects AI to Email MCP Server| AI Agent                 | AI Agent                  | Sticky Note1: "Agent & MCP Client"                                                                            |
| News MCP Client            | `@n8n/n8n-nodes-langchain.mcpClientTool`  | Connects AI to News MCP Server | AI Agent                 | AI Agent                  | Sticky Note1: "Agent & MCP Client"                                                                            |
| Email MCP Server           | `@n8n/n8n-nodes-langchain.mcpTrigger`     | Exposes Gmail tool for email   | Email MCP Client         | Gmail Tool nodes          | Sticky Note2: "Email MCP Server"                                                                               |
| News MCP Server            | `@n8n/n8n-nodes-langchain.mcpTrigger`     | Exposes Tavily and Perplexity  | News MCP Client          | Tavily, Perplexity nodes  | Sticky Note3: "News Research MCP Server"                                                                      |
| Send a message in Gmail    | `n8n-nodes-base.gmailTool`                  | Sends email via Gmail          | Email MCP Server         | None                      | Sticky Note2: "Email MCP Server"                                                                               |
| Send a message in Gmail1   | `n8n-nodes-base.gmailTool`                  | Sends email via Gmail          | Email MCP Server         | None                      | Sticky Note2: "Email MCP Server"                                                                               |
| Send a message in Gmail2   | `n8n-nodes-base.gmailTool`                  | Sends email via Gmail          | Email MCP Server         | None                      | Sticky Note2: "Email MCP Server"                                                                               |
| Search in Tavily           | `@tavily/n8n-nodes-tavily.tavilyTool`      | News search query              | News MCP Server          | News MCP Server           | Sticky Note3: "News Research MCP Server"                                                                      |
| Search in Tavily1          | `@tavily/n8n-nodes-tavily.tavilyTool`      | News search query              | News MCP Server          | News MCP Server           | Sticky Note3: "News Research MCP Server"                                                                      |
| Message a model in Perplexity | `n8n-nodes-base.perplexityTool`          | AI retrieval-augmented answers | News MCP Server          | News MCP Server           | Sticky Note3: "News Research MCP Server"                                                                      |
| Sticky Note                | `n8n-nodes-base.stickyNote`                 | Documentation and instructions| None                     | None                      | Extensive explanation of the workflow, tools, setup, and usage. See detailed description in 1.6.              |
| Sticky Note1               | `n8n-nodes-base.stickyNote`                 | Section label                 | None                     | None                      | "Agent & MCP Client"                                                                                           |
| Sticky Note2               | `n8n-nodes-base.stickyNote`                 | Section label                 | None                     | None                      | "Email MCP Server"                                                                                             |
| Sticky Note3               | `n8n-nodes-base.stickyNote`                 | Section label                 | None                     | None                      | "News Research MCP Server"                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Purpose: To receive user chat messages as input.  
   - Configuration: Default options.

2. **Create the AI Agent Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Configuration:  
     - System Message: `"You are a helpful email assistant.\n\n##Tool\nUse attached Email MCP Tool for emails when asked\n\nUse attached Email MCP Tool for "`  
   - Connect Input from: *When chat message received* node.  
   - Add AI tools and memory as outputs to be connected later.

3. **Create OpenAI Chat Model Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Configuration:  
     - Model: Select `"gpt-4.1-mini"`  
     - Options: Default  
   - Connect Input from: *AI Agent* (as AI language model).  
   - Connect Output to: *AI Agent*.

4. **Create Simple Memory Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Configuration: Default buffer size for recent messages.  
   - Connect Input and Output to *AI Agent* memory ports.

5. **Create MCP Clients for Email and News**  
   - **Email MCP Client**:  
     - Node Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
     - Set `endpointUrl` to your public Email MCP Server endpoint (replace placeholder).  
     - Set `serverTransport` to `"httpStreamable"`.  
     - Connect input/output to *AI Agent* (as AI tool).  

   - **News MCP Client**:  
     - Node Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
     - Set `endpointUrl` to your public News MCP Server endpoint.  
     - Set `serverTransport` to `"httpStreamable"`.  
     - Connect input/output to *AI Agent* (as AI tool).

6. **Create MCP Servers**  
   - **Email MCP Server**:  
     - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
     - Configure `path` with a public, unique path (replace placeholder).  
     - Connect input from *Email MCP Client* tools (Gmail nodes).  

   - **News MCP Server**:  
     - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
     - Configure `path` with a public, unique path (replace placeholder).  
     - Connect input from *News MCP Client* tools (Tavily and Perplexity nodes).

7. **Add Gmail Send Message Nodes (3 variants)**  
   - Node Type: `n8n-nodes-base.gmailTool`  
   - Configure:  
     - `sendTo`, `subject`, and `message` as dynamic fields from AI input.  
     - Connect input from *Email MCP Server* (ai_tool port).  
   - Connect these nodes as outputs of *Email MCP Server*.

8. **Add Tavily Search Nodes (2 variants)**  
   - Node Type: `@tavily/n8n-nodes-tavily.tavilyTool`  
   - Configure `query` parameter to use AI expressions:  
     - `={{ $fromAI('Query', '', 'string') }}`  
   - Connect input from *News MCP Server* (ai_tool port).  
   - Connect outputs back to *News MCP Server*.

9. **Add Perplexity Message Node**  
   - Node Type: `n8n-nodes-base.perplexityTool`  
   - Configure:  
     - `messages.message[0].content` set to `={{ $fromAI('message0_Text', '', 'string') }}`  
     - `simplify` set to `={{ $fromAI('Simplify_Output', '', 'boolean') }}`  
   - Connect input from *News MCP Server* (ai_tool port).  
   - Connect output back to *News MCP Server*.

10. **Add Sticky Notes for Documentation**  
    - Create 4 Sticky Note nodes with the provided content for user guidance and section labeling.  
    - Place them appropriately for clarity.

11. **Credential Setup**  
    - OpenAI: Connect your OpenAI API key to the *OpenAI Chat Model* node.  
    - Gmail: Connect Gmail OAuth2 credentials to all Gmail Tool nodes.  
    - Tavily: Connect Tavily API credentials to Tavily Tool nodes.  
    - Perplexity: Connect Perplexity API credentials to the Perplexity Tool node.

12. **Final Connections**  
    - Ensure all connections mirror the logical flow: chat trigger → AI Agent → OpenAI Chat Model & memory → MCP Clients → MCP Servers → tool nodes for email and news → back to MCP Clients → AI Agent → output.  
    - Validate all placeholder fields (emails, endpoints, paths) have correct production values.

13. **Testing**  
    - Initiate chat messages such as:  
      - “Find today’s top stories on Kubernetes security and draft an intro email to Acme.”  
      - “Summarize the latest AI infra trends and email a 3-bullet update to my team.”  
    - Observe AI responses and email dispatch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Build a chat-first MCP-powered research and outreach assistant combining Tavily, Perplexity, Gmail, and OpenAI with short-term memory for coherent interactions.                                                                                                                                                                                                           | Sticky Note content in workflow                  |
| Watch build-along videos for workflows like these on: www.youtube.com/@automatewithmarc                                                                                                                                                                                                                                                                                        | https://www.youtube.com/@automatewithmarc        |
| Requirements: n8n Cloud or self-hosted instance; OpenAI API key; Tavily, Perplexity, Gmail credentials; publicly reachable MCP endpoints.                                                                                                                                                                                                                                     | Sticky Note content in workflow                  |
| Pro Tip: Use clear action verbs in prompts like “Research X, then email Y with Z takeaways.”                                                                                                                                                                                                                                                                                   | Sticky Note content in workflow                  |
| For safer runs, configure Gmail nodes to send to a test inbox or disable actual send action and only draft emails.                                                                                                                                                                                                                                                            | Sticky Note content in workflow                  |
| Add guardrails in the AI Agent’s system message to align communication tone and style with your brand voice.                                                                                                                                                                                                                                                                   | Sticky Note content in workflow                  |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow built with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.