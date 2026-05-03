The title is excellent, clearly explaining the template's purpose while mentioning all key services.

https://n8nworkflows.xyz/workflows/the-title-is-excellent--clearly-explaining-the-template-s-purpose-while-mentioning-all-key-services--6191


# The title is excellent, clearly explaining the template's purpose while mentioning all key services.

### 1. Workflow Overview

This workflow, named **"Project starter bot"**, automates the creation of structured project folders on Dropbox and notifies relevant teams via Slack and Gmail. It is designed for project coordinators or managers who want to streamline project setup and communication.

The workflow consists of two main logical blocks:

- **1.1 Input Reception and AI-Driven Coordination:**  
  The workflow starts by receiving chat messages from users (e.g., requests to create a new project), processes these inputs through an AI agent configured with project coordination logic, and manages conversation memory.

- **1.2 Folder Management and Notification Execution:**  
  Based on AI agent instructions, the workflow performs folder operations on Dropbox (create, list, move, copy, delete folders and files), then optionally sends notifications to Slack and Gmail once the folders are successfully created and the user confirms.

Supporting these are integration triggers and client nodes for communication with Dropbox, Gmail, Slack, and AI language models (Azure OpenAI and Google Gemini), plus a comprehensive credential setup note.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and AI-Driven Coordination

**Overview:**  
This block captures user chat messages, feeds them to an AI agent configured as an expert project coordinator operating in a two-stage process (folder creation then notification), and maintains conversational context with memory buffers.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- Simple Memory  
- GPT (Azure OpenAI)  
- Gemini (Google Gemini)  

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Entry point for user chat input via webhook.  
  - Configuration: Listens for incoming chat messages to trigger the workflow.  
  - Inputs: External webhook call with chat message payload.  
  - Outputs: Passes message to "AI Agent".  
  - Edge Cases: Webhook availability, malformed input, network timeout.

- **AI Agent**  
  - Type: LangChain Agent node  
  - Role: Core decision-maker executing multi-stage logic as a project coordinator.  
  - Configuration:  
    - System message defines a two-stage process:  
      1. Create project folder and specified subfolders on Dropbox.  
      2. On success, ask user for confirmation to notify team, then send Slack and Gmail notifications.  
    - Includes error handling (stop on folder creation failure, report errors).  
    - Waits for user confirmation before notification stage.  
  - Inputs: Chat messages from trigger, memory data.  
  - Outputs: Commands to Dropbox tools and notification triggers.  
  - Key Expressions: System prompt incorporates variables like project/client name 'X'.  
  - Edge Cases: AI response failures, ambiguous user confirmation, API rate limits.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains sliding window context of conversation for AI Agent.  
  - Configuration: Default buffer settings for short-term memory.  
  - Inputs: AI Agent outputs, chat messages.  
  - Outputs: Contextual memory for AI Agent.  
  - Edge Cases: Memory overflow, loss of context if system restarts.

- **GPT (Azure OpenAI)**  
  - Type: LangChain Language Model Chat AzureOpenAI  
  - Role: Provides AI language model capabilities for AI Agent.  
  - Configuration: Model "motului" deployed on Azure OpenAI service.  
  - Credentials: Azure OpenAI API configured with API key, endpoint, deployment name.  
  - Edge Cases: API key expiry, request timeout, quota exceeded.

- **Gemini (Google Gemini)**  
  - Type: LangChain Language Model Chat Google Gemini  
  - Role: Alternative AI language model for AI Agent (fallback or ensemble).  
  - Configuration: Model "models/gemini-2.5-pro" via Google Palm API.  
  - Credentials: Google API key for Gemini.  
  - Edge Cases: API quota limits, invalid API key, request latency.

---

#### 2.2 Folder Management and Notification Execution

**Overview:**  
This block executes folder operations on Dropbox based on AI Agent commands and sends notifications via Slack and Gmail after user confirmation.

**Nodes Involved:**  
- create folder  
- copy folder  
- move folder  
- delete folder  
- list folder  
- Download a file  
- Upload file  
- Copy file  
- Search file  
- Move file  
- Delete  
- Notify node (MCP Trigger)  
- Notify (MCP Client Tool)  
- Dropbox MCP (MCP Client Tool)  
- Send a message in Slack  
- Send a message in Gmail  

**Node Details:**

- **create folder**  
  - Type: Dropbox Tool (folder creation)  
  - Role: Creates project folders & subfolders on Dropbox as per AI Agent instruction.  
  - Configuration: Path dynamically set via AI Agent variable 'File_Path'.  
  - Authentication: OAuth2 with Dropbox credentials.  
  - Inputs: AI Agent command.  
  - Outputs: Success/failure status to AI Agent.  
  - Edge Cases: Folder already exists, permission denied, API failure.

- **copy folder**  
  - Type: Dropbox Tool (folder copy)  
  - Role: Copies folders within Dropbox as needed.  
  - Configuration: Source and target paths from AI Agent variables.  
  - Authentication: OAuth2 Dropbox.  
  - Edge Cases: Source path missing, destination path exists.

- **move folder**  
  - Type: Dropbox Tool (folder copy operation used as move)  
  - Role: Moves folders within Dropbox.  
  - Configuration: Paths from AI Agent inputs.  
  - Edge Cases: Similar to copy folder.

- **delete folder**  
  - Type: Dropbox Tool (folder deletion)  
  - Role: Deletes folders as requested by AI Agent error handling or cleanup.  
  - Edge Cases: Folder not found, permission issues.

- **list folder**  
  - Type: Dropbox Tool (folder listing)  
  - Role: Retrieves folder contents for verification or processing.  
  - Edge Cases: Path not found, empty folders.

- **Download a file**  
  - Type: Dropbox Tool (file download)  
  - Role: Downloads a file from Dropbox if needed.  
  - Edge Cases: File not found, download failure.

- **Upload file**  
  - Type: Dropbox Tool (file upload)  
  - Role: Uploads files to Dropbox.  
  - Edge Cases: File size limits, upload errors.

- **Copy file**, **Move file**, **Delete**  
  - Type: Dropbox Tool (file operations)  
  - Role: Perform respective file management tasks requested by AI Agent.  
  - Edge Cases: Same as folder operations.

- **Search file**  
  - Type: Dropbox Tool (search)  
  - Role: Searches Dropbox for files/folders matching query strings.  
  - Edge Cases: No results found, query syntax errors.

- **Notify node (MCP Trigger)**  
  - Type: LangChain MCP Trigger  
  - Role: Receives triggers for notification sending stage.  
  - Inputs: Confirmation from AI Agent to notify team.  
  - Outputs: Activates notification client nodes.

- **Notify (MCP Client Tool)**  
  - Type: LangChain MCP Client Tool  
  - Role: Sends real-time notifications or commands for Slack and Gmail messages.  
  - Configuration: SSE endpoint configured for notification channel.  
  - Edge Cases: SSE connection loss.

- **Dropbox MCP (MCP Client Tool)**  
  - Type: LangChain MCP Client Tool  
  - Role: Facilitates communication between AI Agent and Dropbox operations.  
  - Edge Cases: Connection issues.

- **Send a message in Slack**  
  - Type: Slack Tool  
  - Role: Posts confirmation message to Slack #projects channel after folder creation.  
  - Configuration: Channel URL hardcoded, message from AI Agent variables.  
  - Credentials: Slack OAuth Bot token.  
  - Edge Cases: Bot permissions, invalid channel.

- **Send a message in Gmail**  
  - Type: Gmail Tool  
  - Role: Sends email notification confirming project setup.  
  - Configuration: Recipient, subject, and message dynamically set from AI Agent variables.  
  - Credentials: Gmail OAuth2 credentials.  
  - Edge Cases: Authentication failure, invalid email address.

---

#### 2.3 Supporting Credential Setup Note

**Overview:**  
Provides detailed instructions for setting up required OAuth2 credentials for Dropbox, Gmail, Slack, Azure OpenAI, and Google Gemini.

**Node Involved:**  
- Sticky Note  

**Node Details:**

- **Sticky Note**  
  - Type: Static informational node  
  - Content: Step-by-step credential setup guides with external links to developer consoles and API portals.  
  - Use: Reference for administrators configuring the integrations.  
  - Important Notes: Redirect URIs, API scopes, and permissions clearly outlined.  
  - Edge Cases: Misconfigured credentials will cause authentication failures in various nodes.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                          | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                                                 |
|----------------------------|--------------------------------------|----------------------------------------|--------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger                | Entry point for user chat input        | -                              | AI Agent                        |                                                                                                                             |
| AI Agent                   | LangChain Agent                      | Core AI logic for project coordination | When chat message received, GPT, Gemini, Simple Memory | MCP Server Trigger, Dropbox MCP, Notify |                                                                                                                             |
| Simple Memory              | LangChain Memory Buffer Window       | Maintains conversation context         | AI Agent                       | AI Agent                       |                                                                                                                             |
| GPT                        | LangChain LM Chat AzureOpenAI        | AI language model for Agent             | -                              | AI Agent                       |                                                                                                                             |
| Gemini                     | LangChain LM Chat Google Gemini      | Alternative AI language model           | -                              | AI Agent                       |                                                                                                                             |
| MCP Server Trigger         | LangChain MCP Trigger                | Receives AI Agent tool commands         | AI Agent                      | Delete, Copy file, Move file, ... |                                                                                                                             |
| create folder              | Dropbox Tool                        | Creates project folders on Dropbox      | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| copy folder                | Dropbox Tool                        | Copies folders on Dropbox                | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| move folder                | Dropbox Tool                        | Moves folders on Dropbox                 | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| delete folder              | Dropbox Tool                        | Deletes folders on Dropbox               | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| list folder                | Dropbox Tool                        | Lists folder contents                    | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| Download a file            | Dropbox Tool                        | Downloads files from Dropbox             | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| Upload file                | Dropbox Tool                        | Uploads files to Dropbox                 | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| Copy file                  | Dropbox Tool                        | Copies files on Dropbox                   | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| Move file                  | Dropbox Tool                        | Moves files on Dropbox                    | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| Delete                     | Dropbox Tool                        | Deletes files on Dropbox                  | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| Search file                | Dropbox Tool                        | Searches files/folders on Dropbox        | MCP Server Trigger             | MCP Server Trigger             |                                                                                                                             |
| Notify node                | LangChain MCP Trigger                | Triggers notification sending            | AI Agent                      | Notify                        |                                                                                                                             |
| Notify                     | LangChain MCP Client Tool            | Sends notification commands              | Notify node                   | Send a message in Slack, Send a message in Gmail |                                                                                                                             |
| Dropbox MCP                | LangChain MCP Client Tool            | Connects AI Agent with Dropbox operations| AI Agent                      | MCP Server Trigger             |                                                                                                                             |
| Send a message in Slack    | Slack Tool                         | Posts notification message to Slack     | Notify                        | -                             |                                                                                                                             |
| Send a message in Gmail    | Gmail Tool                         | Sends notification email to user        | Notify                        | -                             |                                                                                                                             |
| Sticky Note                | Sticky Note                        | Credential setup instructions            | -                              | -                             | # 🔑 Credentials Setup Guide ... [see detailed content in section 2.3]                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: LangChain Chat Trigger  
   - Purpose: Listen for incoming chat messages via webhook (generate webhook URL).  
   - No special parameters.

2. **Create "AI Agent" node**  
   - Type: LangChain Agent  
   - Configure system message with detailed two-stage project coordination logic (folder creation → notification after confirmation).  
   - Enable fallback for robustness.  
   - Link "When chat message received" output to this node's input.

3. **Create "Simple Memory" node**  
   - Type: LangChain Memory Buffer Window  
   - Default settings to maintain conversation context.  
   - Connect output of "AI Agent" to "Simple Memory" and back to "AI Agent" memory input.

4. **Create AI language model nodes "GPT" and "Gemini"**  
   - GPT: Use LangChain LM Chat AzureOpenAI node, set model to your Azure deployment name (e.g., "motului").  
   - Gemini: Use LangChain LM Chat Google Gemini node, set model name to "models/gemini-2.5-pro".  
   - Connect both to "AI Agent" as languageModel inputs.

5. **Set up Dropbox folder operation nodes:**  
   - Create multiple Dropbox Tool nodes for folder and file operations: `create folder`, `copy folder`, `move folder`, `delete folder`, `list folder`, `Download a file`, `Upload file`, `Copy file`, `Move file`, `Delete`, `Search file`.  
   - Configure each node with dynamic path parameters using AI Agent variables (e.g., `{{$fromAI('File_Path')}}`).  
   - Authenticate each with Dropbox OAuth2 credentials (set up per sticky note instructions).

6. **Create MCP Server Trigger node ("MCP Server Trigger")**  
   - Type: LangChain MCP Trigger  
   - Setup webhook path (unique) to receive commands from AI Agent for Dropbox operations.  
   - Connect outputs to Dropbox operation nodes.

7. **Create MCP Client Tool nodes:**  
   - "Dropbox MCP": LangChain MCP Client Tool connected to AI Agent for folder ops.  
   - "Notify": LangChain MCP Client Tool connected to Notify node for notification commands.

8. **Create Notify-related nodes:**  
   - "Notify node": LangChain MCP Trigger to receive notification triggers.  
   - "Send a message in Slack": Slack Tool node, configure with channel ID for #projects, OAuth token for Slack Bot.  
   - "Send a message in Gmail": Gmail Tool node, configure with OAuth2 credentials, dynamic recipient, subject, and message from AI Agent variables.

9. **Link notification flow:**  
   - Connect "Notify node" to "Notify" MCP Client Tool.  
   - Connect "Notify" outputs to Slack and Gmail nodes.

10. **Add Sticky Note node**  
    - Add a sticky note with detailed credential setup instructions and important tips for Dropbox, Gmail, Slack, Azure OpenAI, and Google Gemini credentials.

11. **Configure credentials:**  
    - Set up OAuth2 credentials for Dropbox, Gmail, Slack as per sticky note instructions.  
    - Set up API keys and endpoints for Azure OpenAI and Google Gemini.  
    - Test each credential for access and quota.

12. **Test the workflow:**  
    - Send a chat message via webhook to trigger project creation.  
    - Verify folder creation on Dropbox with subfolders.  
    - Confirm notification prompt works and sends Slack and Gmail messages upon approval.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| 🔑 Credentials Setup Guide for Dropbox OAuth2, Gmail OAuth2, Slack API, Azure OpenAI, and Google Gemini. Detailed step-by-step instructions with links to developer portals and required scopes/redirect URIs. Includes tips on testing and secure storage.                                                                                                                                                                                                                                                             | See Sticky Note node in workflow for full content: https://www.dropbox.com/developers/apps, https://console.cloud.google.com/, https://api.slack.com/apps, https://portal.azure.com/, https://aistudio.google.com/ |
| This workflow uses LangChain nodes for AI integration and MCP (Message Control Protocol) triggers and clients for real-time tool invocation and notifications.                                                                                                                                                                                                                                                                                                                                                         | n8n LangChain and MCP node documentation                                                                |
| Slack notification is hardcoded to a specific channel URL; adapt channel ID as needed for your workspace. Gmail messages use dynamic recipient and subject fields supplied by AI Agent.                                                                                                                                                                                                                                                                                                                               | Slack API and Gmail OAuth2 configuration                                                                |
| Folder structure created by AI Agent is fixed with 5 subfolders for project management best practices: Briefs_and_Contracts, Source_Assets, Work_In_Progress, Internal_Review, Final_Deliverables. Modify system message in AI Agent node to change this.                                                                                                                                                                                                                                                                | AI Agent system prompt                                                                                   |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, a workflow automation tool. This processing fully complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.