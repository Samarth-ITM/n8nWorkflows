Automated Email Assistant for Suppliers using OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/automated-email-assistant-for-suppliers-using-openai-and-google-sheets-4067


# Automated Email Assistant for Suppliers using OpenAI and Google Sheets

### 1. Workflow Overview

This workflow implements an **Automated Email Assistant for Suppliers** using n8n with OpenAI and Google Sheets integration. It is designed to facilitate sending personalized emails to suppliers by automatically resolving supplier contact information and generating email content through an AI assistant.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Triggering:** Receives incoming chat or message requests via MCP Server or Chat Trigger, serving as entry points for user interaction.
- **1.2 AI Processing and Decision Making:** Uses an AI agent powered by OpenAI to analyze requests, think through the required information, consult supplier data in Google Sheets if needed, and prepare email content.
- **1.3 Email Dispatch:** Sends the generated email using Gmail, interfacing with the MCP server for message delivery confirmation.

This design enables a seamless, automated assistant experience that can interpret natural language requests, retrieve required data, and execute actions with minimal manual intervention.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

**Overview:**  
Handles incoming requests either from a dedicated MCP Server webhook or a chat interface, initiating the workflow.

**Nodes Involved:**  
- MCP Server Trigger  
- Entry Point: Chat with the Agent

**Node Details:**

- **MCP Server Trigger**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: Listens for incoming HTTP webhook requests from the MCP Server to start the workflow.  
  - Configurations: Path set as a unique webhook ID for identification.  
  - Inputs: External HTTP request (MCP server).  
  - Outputs: Triggers downstream AI processing nodes.  
  - Edge Cases: Webhook connection errors, incorrect payload format, network timeouts.

- **Entry Point: Chat with the Agent**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Receives chat-based input to initiate the AI assistant workflow.  
  - Configurations: Default chat trigger options.  
  - Inputs: Chat messages from users.  
  - Outputs: Passes data to the AI agent node.  
  - Edge Cases: Chat connection issues, malformed user input.

---

#### 2.2 AI Processing and Decision Making

**Overview:**  
Processes user input through an AI agent that uses OpenAI's GPT-4o-mini model for natural language understanding, consults a Google Sheets supplier database for missing contact information, and prepares the email content.

**Nodes Involved:**  
- Personal Email Assistant  
- AI Model Open AI  
- Chat Memory  
- Thinker, think before executing.  
- Supplier database.  
- Node that connects to the MCP server.

**Node Details:**

- **Personal Email Assistant**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Core AI agent orchestrating the workflow steps, integrating tools and driving logic.  
  - Configuration:  
    - System message instructs the assistant to:  
      1) Use the “Think” tool to analyze request completeness.  
      2) Use current date/time in America/Mexico_City timezone.  
      3) Query Google Sheets for supplier email if missing.  
      4) Perform requested actions and respond clearly.  
      5) Request missing info if needed.  
  - Inputs: Receives chat or MCP server trigger data.  
  - Outputs: Sends commands to tools like “Thinker”, Google Sheets, and MCP server node.  
  - Edge Cases: AI misinterpretation, missing or ambiguous user data, API rate limits.

- **AI Model Open AI**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: Provides the GPT-4o-mini language model for AI reasoning and response generation.  
  - Configuration: Model set to "gpt-4o-mini", connected with OpenAI credentials.  
  - Inputs: Prompt and context from the agent node.  
  - Outputs: Language model text completions to the agent.  
  - Edge Cases: API quota exceeded, network errors, malformed prompts.

- **Chat Memory**  
  - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - Role: Maintains a sliding window of conversation context (20 messages) for coherent multi-turn dialogue.  
  - Configuration: Context window length set to 20.  
  - Inputs: Conversation messages from the AI agent.  
  - Outputs: Context to the AI model.  
  - Edge Cases: Memory overflow, context loss on restart.

- **Thinker, think before executing.**  
  - Type: `@n8n/n8n-nodes-langchain.toolThink`  
  - Role: Analytical tool node that helps the AI agent verify if all required information is available before proceeding.  
  - Inputs: Message content from the AI agent.  
  - Outputs: Insight or confirmation to the AI agent.  
  - Edge Cases: Logic errors, incomplete analysis.

- **Supplier database.**  
  - Type: `n8n-nodes-base.googleSheetsTool`  
  - Role: Searches Google Sheets “Suppliers” document for supplier contact emails based on supplier name.  
  - Configuration:  
    - Document ID and sheet name set to a specific Google Sheets containing supplier data.  
    - Filter uses a lookup column “Nombre del Proveedor” with dynamic lookup value from AI.  
    - Uses Google Sheets OAuth2 credentials.  
  - Inputs: Supplier name from AI agent.  
  - Outputs: Supplier contact email(s) back to AI agent.  
  - Edge Cases: Missing Google Sheets permissions, empty results, network issues.

- **Node that connects to the MCP server.**  
  - Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
  - Role: Sends data back to the MCP server, supporting server-sent events (SSE) for message streaming.  
  - Configuration: SSE endpoint URL provided for real-time communication.  
  - Inputs: AI agent's final commands/messages.  
  - Outputs: MCP server receives messages for further processing or display.  
  - Edge Cases: SSE connection failure, timeouts.

---

#### 2.3 Email Dispatch

**Overview:**  
Executes the actual sending of emails through Gmail based on the AI-generated content and resolved recipient addresses.

**Nodes Involved:**  
- Send message by Gmail

**Node Details:**

- **Send message by Gmail**  
  - Type: `n8n-nodes-base.gmailTool`  
  - Role: Sends plain-text emails to the resolved supplier contacts.  
  - Configuration:  
    - Recipient address, subject, and message body dynamically injected from AI-generated data.  
    - Uses Gmail OAuth2 credentials for authentication.  
  - Inputs: Email details from MCP Server Trigger and AI agent.  
  - Outputs: Confirmation of email sent.  
  - Edge Cases: Authentication failure, invalid email addresses, Gmail API rate limits.

---

### 3. Summary Table

| Node Name                      | Node Type                                         | Functional Role                          | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                             |
|--------------------------------|--------------------------------------------------|----------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------|
| MCP Server Trigger             | @n8n/n8n-nodes-langchain.mcpTrigger              | Receives HTTP webhook requests          | External webhook              | Send message by Gmail        | ## To implement this personal email assistant for suppliers, you need to create a Google Sheets file named "Suppliers" with two columns: Supplier Name, Contact Email. This is the MCP Server. |
| Send message by Gmail          | n8n-nodes-base.gmailTool                          | Sends the email                         | MCP Server Trigger            |                             |                                                                                                       |
| Sticky Note                   | n8n-nodes-base.stickyNote                         | Instructional note                      |                              |                             | ## To implement this personal email assistant for suppliers, you need to follow steps involving Google Sheets and MCP Server.              |
| Sticky Note1                  | n8n-nodes-base.stickyNote                         | Instructional note                      |                              |                             | ## This is the AI Agent connecting to the MCP Server (MCP Client). Steps include receiving message, thinking, searching Google Sheets, and sending email. |
| Entry Point: Chat with the Agent | @n8n/n8n-nodes-langchain.chatTrigger             | Chat input trigger                      | External chat input           | Personal Email Assistant     |                                                                                                       |
| Personal Email Assistant      | @n8n/n8n-nodes-langchain.agent                    | AI agent orchestrating assistant logic | Entry Point: Chat with the Agent | AI Model Open AI, Chat Memory, Thinker, Supplier database., Node that connects to the MCP server. |                                                                                                       |
| AI Model Open AI              | @n8n/n8n-nodes-langchain.lmChatOpenAi             | Provides AI language model              | Personal Email Assistant      | Personal Email Assistant     |                                                                                                       |
| Chat Memory                  | @n8n/n8n-nodes-langchain.memoryBufferWindow       | Maintains conversation context         | Personal Email Assistant      | AI Model Open AI             |                                                                                                       |
| Thinker, think before executing. | @n8n/n8n-nodes-langchain.toolThink                | Analyzes completeness of request       | Personal Email Assistant      | Personal Email Assistant     |                                                                                                       |
| Supplier database.           | n8n-nodes-base.googleSheetsTool                   | Looks up supplier email in Google Sheets | Personal Email Assistant      | Personal Email Assistant     |                                                                                                       |
| Node that connects to the MCP server. | @n8n/n8n-nodes-langchain.mcpClientTool             | Sends data back to MCP server           | Personal Email Assistant      | Personal Email Assistant     |                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger node:**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Configure a unique webhook path.  
   - This node receives HTTP requests from the MCP server to start the workflow.

2. **Create Entry Point: Chat with the Agent node:**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Default settings for chat input.  
   - Acts as a chat-based entry point.

3. **Create Personal Email Assistant node:**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Configure system message with instructions for AI:  
     - Use Think tool to analyze requests.  
     - Use current date/time in America/Mexico_City timezone.  
     - Lookup supplier emails in Google Sheets if missing.  
     - Perform requested actions and respond clearly.  
     - Request missing info if needed.

4. **Create AI Model Open AI node:**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Select model: `gpt-4o-mini`.  
   - Attach OpenAI API credentials.

5. **Create Chat Memory node:**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set context window length to 20.

6. **Create Thinker node:**  
   - Type: `@n8n/n8n-nodes-langchain.toolThink`  
   - No special configuration required.

7. **Create Supplier database node:**  
   - Type: `n8n-nodes-base.googleSheetsTool`  
   - Connect to Google Sheets OAuth2 credentials.  
   - Set Document ID to your "Suppliers" Google Sheets file (must exist).  
   - Set Sheet Name to the first sheet (gid=0).  
   - Configure filter: lookup column "Nombre del Proveedor" with dynamic lookup value from AI agent.

8. **Create Node that connects to the MCP server:**  
   - Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
   - Set SSE endpoint URL to your MCP server SSE endpoint.

9. **Create Send message by Gmail node:**  
   - Type: `n8n-nodes-base.gmailTool`  
   - Configure Gmail OAuth2 credentials.  
   - Set:  
     - Send To: dynamic email from AI agent or Google Sheets lookup.  
     - Subject: dynamic subject from AI agent.  
     - Message: dynamic message body from AI agent.  
     - Email Type: text.

10. **Connect nodes as follows:**  
    - MCP Server Trigger → Send message by Gmail  
    - Entry Point: Chat with the Agent → Personal Email Assistant  
    - Personal Email Assistant → AI Model Open AI, Chat Memory, Thinker, Supplier database, Node that connects to MCP server  
    - Chat Memory → AI Model Open AI  
    - AI Model Open AI → Personal Email Assistant  
    - Thinker → Personal Email Assistant  
    - Supplier database → Personal Email Assistant  
    - Node that connects to MCP server → Personal Email Assistant

11. **Create Google Sheets document "Suppliers":**  
    - Columns: Supplier Name, Contact Email  
    - Populate with supplier data.

12. **Ensure all credentials (OpenAI, Gmail, Google Sheets) are properly set up and authorized.**

13. **Test workflow by sending messages through the MCP Server webhook or chat interface.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Google Sheets document "Suppliers" is mandatory with columns "Supplier Name" and "Contact Email" for email lookup functionality to work correctly.   | Google Sheets: https://docs.google.com/spreadsheets/d/1vFkApkyOU1npZJvl90sIZ4x8xnSXylfTjvCGhWTeOB8 |
| MCP Server SSE endpoint must be active and accessible for real-time communication with the MCP client node.                                           | SSE Endpoint URL visible in node configuration.                                                    |
| The AI assistant uses timezone America/Mexico_City for current date/time in system messages.                                                         | Timezone configuration inside Personal Email Assistant node system message.                        |
| Gmail OAuth2 credentials require proper permission for sending emails on behalf of the user.                                                          | Gmail API documentation: https://developers.google.com/gmail/api                                     |
| OpenAI API key associated with GPT-4o-mini model must be valid and have sufficient quota.                                                             | OpenAI API documentation: https://platform.openai.com/docs/models/gpt-4o-mini                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.