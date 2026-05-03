Build a Personal Assistant with Google Gemini, Gmail and Calendar using MCP

https://n8nworkflows.xyz/workflows/build-a-personal-assistant-with-google-gemini--gmail-and-calendar-using-mcp-3905


# Build a Personal Assistant with Google Gemini, Gmail and Calendar using MCP

### 1. Workflow Overview

This workflow implements a **Personal Assistant MCP server** that enables natural language interaction with various applications such as Google Calendar, Gmail, and Google Sheets (used here as a CRM). It leverages the **Google Gemini AI model** for natural language understanding and decision making, combined with the **Model Context Protocol (MCP)** to select and orchestrate automated tasks across integrated apps.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Receives user chat messages via a LangChain Chat Trigger node.
- **1.2 AI Brain Processing:** Uses Google Gemini chat model and a memory buffer to comprehend and retain conversational context.
- **1.3 AI Agent Decision:** An agent node that interprets the user's intent and selects the appropriate tool to execute.
- **1.4 MCP System:** Includes the MCP Server Trigger and MCP Client nodes that manage communication between the AI agent and connected app tools.
- **1.5 App Integration Tools:** Nodes for interacting with Google Calendar, Gmail, and Google Sheets to perform specific tasks such as creating/updating calendar events, managing emails, and manipulating CRM contacts.
- **1.6 Memory Handling:** The Simple Memory node buffers conversation context, enabling coherent multi-turn conversations.
- **1.7 Sticky Notes:** Documentation nodes providing usage instructions and examples for the calendar, email, and CRM nodes to guide users in extending or customizing the workflow.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block captures user input from a chat interface, serving as the entry point for the personal assistant workflow.

**Nodes Involved:**  
- When chat message received

**Node Details:**  

| Node Name                | Details                                                                                         |
|--------------------------|------------------------------------------------------------------------------------------------|
| When chat message received| - Type: LangChain Chat Trigger node                                                         |
|                          | - Role: Listens for incoming chat messages to trigger the workflow                            |
|                          | - Config: Default options, webhook enabled with ID "989c3a79-5a0c-4ca1-a542-55e060816121"     |
|                          | - Output: Sends the received message data downstream to the AI Agent node                     |
|                          | - Possible errors: Webhook connection failures, missing payload, unauthorized webhook calls   |

---

#### 1.2 AI Brain Processing

**Overview:**  
This block processes the user input using the Google Gemini chat model and manages conversation state via a memory buffer.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Simple Memory

**Node Details:**  

| Node Name               | Details                                                                                          |
|-------------------------|-------------------------------------------------------------------------------------------------|
| Google Gemini Chat Model | - Type: LangChain Google Gemini Chat Model node                                                 |
|                         | - Role: Sends user input to Google Gemini model (Gemini 2.5 Pro preview) for natural language understanding |
|                         | - Configuration: Model set to "models/gemini-2.5-pro-preview-05-06"                             |
|                         | - Credentials: Uses Google Palm API account                                                     |
|                         | - Input: Receives user messages from AI Agent node                                             |
|                         | - Output: Provides model's interpretation and response to AI Agent                             |
|                         | - Potential failures: API key errors, rate limits, network timeouts, model unavailability      |
| Simple Memory           | - Type: LangChain Memory Buffer Window node                                                     |
|                         | - Role: Maintains recent conversational context (last 5 interactions by default)               |
|                         | - Config: Default parameters, no custom window size specified                                  |
|                         | - Input: Connected as AI memory to the AI Agent                                                |
|                         | - Output: Supplies conversation context back to AI Agent                                       |
|                         | - Edge cases: Memory overflow, data loss on restart, failures to retrieve context              |

---

#### 1.3 AI Agent Decision

**Overview:**  
This block represents the core assistant logic that interprets the user’s intent and decides which tool or app node to invoke.

**Nodes Involved:**  
- Personal Assistant (LangChain Agent)

**Node Details:**  

| Node Name           | Details                                                                                         |
|---------------------|------------------------------------------------------------------------------------------------|
| Personal Assistant   | - Type: LangChain Agent node                                                                    |
|                     | - Role: Acts as the assistant's decision maker, interpreting user requests and selecting tools  |
|                     | - Config: Default options, integrates with memory and language model nodes                      |
|                     | - Inputs: Receives chat messages from "When chat message received", language model output, and memory context |
|                     | - Outputs: Sends tool invocation commands to MCP Client node                                   |
|                     | - Potential failure points: Expression errors, improper tool mapping, timeout on decisions      |

---

#### 1.4 MCP System

**Overview:**  
Manages the communication between the AI agent and the app-specific nodes using the Model Context Protocol (MCP).

**Nodes Involved:**  
- MCP Server Trigger  
- MCP Client

**Node Details:**  

| Node Name          | Details                                                                                         |
|--------------------|------------------------------------------------------------------------------------------------|
| MCP Server Trigger | - Type: LangChain MCP Trigger node                                                             |
|                    | - Role: Listens for requests from AI agent to trigger app tool workflows                        |
|                    | - Config: Webhook path set to "b37ab045-0b99-4d57-af44-6ae1e9ac6381"                           |
|                    | - Input: Receives commands from MCP Client                                                     |
|                    | - Output: Routes commands to appropriate app integration nodes                                 |
|                    | - Errors: Webhook failures, incorrect path, security/auth issues                               |
| MCP Client         | - Type: LangChain MCP Client Tool                                                               |
|                    | - Role: Sends tool execution commands from AI Agent to MCP Server Trigger                      |
|                    | - Config: Requires user to set the SSE endpoint URL for MCP server                             |
|                    | - Input: Receives AI Agent output                                                              |
|                    | - Output: Forwards commands to MCP Server Trigger                                              |
|                    | - Failure cases: Missing or incorrect SSE endpoint, connection timeouts                        |

---

#### 1.5 App Integration Tools

**Overview:**  
This block includes all nodes that interact directly with external apps (Google Calendar, Gmail, Google Sheets) to perform user-requested actions.

**Nodes Involved:**  
- Google Calendar nodes: Create event, Update event, Find single event, Find multiple events  
- Gmail nodes: Draft email, Find emails  
- Google Sheets nodes (CRM): Add new row, Find row, Update row

**Node Details:**  

| Node Name        | Details                                                                                                         |
|------------------|----------------------------------------------------------------------------------------------------------------|
| Create event     | - Type: Google Calendar Tool                                                                                     |
|                  | - Role: Creates a new event in the specified Google Calendar                                                   |
|                  | - Config: Calendar set to "hello@1node.ai" with default additional fields                                      |
|                  | - Credentials: Google Calendar OAuth2 account                                                                   |
|                  | - Input: MCP Server Trigger commands                                                                            |
|                  | - Failure cases: Invalid calendar ID, permission denied, API quota exceeded                                     |
| Update event     | - Type: Google Calendar Tool                                                                                     |
|                  | - Role: Updates an existing calendar event based on Event_ID parameter                                         |
|                  | - Config: Calendar set to "hello@1node.ai", operation "update"                                                  |
|                  | - Credentials: Google Calendar OAuth2 account                                                                   |
|                  | - Input: MCP Server Trigger commands                                                                            |
|                  | - Edge cases: Non-existent event ID, API errors                                                                |
| Find single event| - Type: Google Calendar Tool                                                                                     |
|                  | - Role: Retrieves details of a specific event by Event_ID                                                      |
|                  | - Config: Calendar "hello@1node.ai", operation "get"                                                            |
|                  | - Credentials: Google Calendar OAuth2 account                                                                   |
|                  | - Input: MCP Server Trigger                                                                                      |
| Find multiple events| - Type: Google Calendar Tool                                                                                   |
|                    | - Role: Fetches multiple events (up to limit 10) from calendar                                                |
|                    | - Config: Calendar "hello@1node.ai", operation "getAll"                                                        |
|                    | - Credentials: Google Calendar OAuth2 account                                                                   |
| Draft email      | - Type: Gmail Tool                                                                                                |
|                  | - Role: Drafts an email message in Gmail                                                                        |
|                  | - Config: Draft resource used, subject and message content dynamically populated via AI agent overrides         |
|                  | - Credentials: Gmail OAuth2                                                                                      |
|                  | - Input: MCP Server Trigger                                                                                      |
| Find emails      | - Type: Gmail Tool                                                                                                |
|                  | - Role: Searches and retrieves emails from Gmail inbox based on filters                                        |
|                  | - Config: Operation "getAll", "Return_All" parameter dynamically set by AI agent                               |
|                  | - Credentials: Gmail OAuth2                                                                                      |
| Add new row      | - Type: Google Sheets Tool                                                                                        |
|                  | - Role: Appends a new row to the Google Sheet used as CRM                                                       |
|                  | - Config: Document and sheet ID set to a specific Google Sheet (Contacts)                                       |
|                  | - Credentials: Google Sheets OAuth2                                                                              |
| Find row        | - Type: Google Sheets Tool                                                                                        |
|                  | - Role: Finds a matching row in Google Sheet                                                                    |
|                  | - Config: Same sheet & document as Add new row                                                                  |
|                  | - Credentials: Google Sheets OAuth2                                                                              |
| Update row      | - Type: Google Sheets Tool                                                                                        |
|                  | - Role: Updates a row in Google Sheet identified by row_number                                                  |
|                  | - Config: Uses "row_number" as matching column                                                                   |
|                  | - Credentials: Google Sheets OAuth2                                                                              |

---

#### 1.6 Memory Handling

**Overview:**  
The Simple Memory node buffers recent interaction context, linked to the AI Agent node to maintain conversational continuity.

**Nodes Involved:**  
- Simple Memory (also described in 1.2)

**Node Details:**  
See section 1.2 for details.

---

#### 1.7 Sticky Notes

**Overview:**  
Documentation and guidance nodes embedded in the workflow for user reference about how to use calendar, email, and CRM nodes effectively.

**Nodes Involved:**  
- Sticky Note (Calendar nodes)  
- Sticky Note1 (Email nodes)  
- Sticky Note2 (CRM nodes)  
- Sticky Note3 (MCP Client notes)  
- Sticky Note4 (Empty/placeholder)

**Node Details:**  

| Node Name    | Details                                                                                              |
|--------------|-----------------------------------------------------------------------------------------------------|
| Sticky Note  | Explains calendar node capabilities and gives multi-node usage examples                             |
| Sticky Note1 | Explains email node usage scenarios and examples                                                    |
| Sticky Note2 | Describes CRM Google Sheets nodes and example commands                                             |
| Sticky Note3 | Details about MCP Client setup instructions and output customization                               |
| Sticky Note4 | Empty note, no content                                                                              |

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                      | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                          |
|-------------------------|----------------------------------|------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------|
| When chat message received| LangChain Chat Trigger           | Input reception                    |                             | Personal Assistant          |                                                                                                                      |
| Google Gemini Chat Model | LangChain Google Gemini Chat     | AI language model                  | Personal Assistant           | Personal Assistant          |                                                                                                                      |
| Simple Memory            | LangChain Memory Buffer Window   | Conversation context memory       | Personal Assistant           | Personal Assistant          |                                                                                                                      |
| MCP Server Trigger       | LangChain MCP Trigger            | MCP command trigger               | MCP Client; Find row; Update row; Add new row; Draft email; Find emails; Create event; Update event; Find single event; Find multiple events | Various app nodes           |                                                                                                                      |
| MCP Client               | LangChain MCP Client Tool        | Sends AI agent tool commands      | Personal Assistant           | MCP Server Trigger          | Paste your MCP client URL from the MCP server trigger node. Customize your output node to receive workflow completion notifications (eg. Telegram, Gmail) from your personal assistant |
| Create event             | Google Calendar Tool             | Create calendar event             | MCP Server Trigger           |                            | # Calendar nodes - You could order your agent to create a new event in your Google Calendar, find, update events, etc. Examples included in sticky note.                |
| Update event             | Google Calendar Tool             | Update calendar event             | MCP Server Trigger           |                            | # Calendar nodes (see above)                                                                                         |
| Find single event        | Google Calendar Tool             | Retrieve single event details     | MCP Server Trigger           |                            | # Calendar nodes (see above)                                                                                         |
| Find multiple events     | Google Calendar Tool             | Retrieve multiple events          | MCP Server Trigger           |                            | # Calendar nodes (see above)                                                                                         |
| Draft email              | Gmail Tool                      | Draft an email                   | MCP Server Trigger           |                            | # Email nodes - Search inbox, draft replies, etc. Examples included in sticky note.                                   |
| Find emails              | Gmail Tool                      | Search emails                   | MCP Server Trigger           |                            | # Email nodes (see above)                                                                                            |
| Add new row              | Google Sheets Tool              | Add a new CRM contact             | MCP Server Trigger           |                            | # CRM nodes - Add, find, update contact data examples included in sticky note.                                       |
| Find row                 | Google Sheets Tool              | Find CRM contact details          | MCP Server Trigger           |                            | # CRM nodes (see above)                                                                                              |
| Update row               | Google Sheets Tool              | Update CRM contact info           | MCP Server Trigger           |                            | # CRM nodes (see above)                                                                                              |
| Personal Assistant       | LangChain Agent                 | AI agent decision logic          | When chat message received; Google Gemini Chat Model; Simple Memory | MCP Client                 |                                                                                                                      |
| Sticky Note              | Sticky Note node                | Documentation on calendar nodes  |                             |                            | # Calendar nodes documentation                                                                                        |
| Sticky Note1             | Sticky Note node                | Documentation on email nodes     |                             |                            | # Email nodes documentation                                                                                           |
| Sticky Note2             | Sticky Note node                | Documentation on CRM nodes       |                             |                            | # CRM nodes documentation                                                                                             |
| Sticky Note3             | Sticky Note node                | Documentation on MCP client setup|                             |                            | Paste your MCP client URL from the MCP server trigger node. Customize your output node to receive workflow completion notifications. |
| Sticky Note4             | Sticky Note node                | Empty/placeholder                |                             |                            |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: When chat message received**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Set webhook enabled (auto-generated ID)  
   - Default options

2. **Create AI Language Model Node: Google Gemini Chat Model**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Model name: `models/gemini-2.5-pro-preview-05-06`  
   - Credentials: Assign Google Palm API credentials with valid API key

3. **Create Memory Node: Simple Memory**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Default parameters (5 interaction window)  
   - No special config needed

4. **Create AI Agent Node: Personal Assistant**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Default options  
   - Connect inputs:  
     - From "When chat message received" (main)  
     - AI language model from "Google Gemini Chat Model"  
     - AI memory from "Simple Memory"

5. **Create MCP Server Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Webhook path: set to a unique string (e.g., `b37ab045-0b99-4d57-af44-6ae1e9ac6381`)  
   - No special config

6. **Create MCP Client Node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
   - Set `sseEndpoint` parameter to your MCP server URL (must match MCP Server Trigger webhook)  
   - Connect input from "Personal Assistant" node

7. **Connect MCP Server Trigger outputs to each app integration node below**

8. **Create Google Calendar Nodes:**  
   - **Create event:**  
     - Type: `n8n-nodes-base.googleCalendarTool`  
     - Operation: Create event (default)  
     - Calendar: select your calendar email (e.g., `hello@1node.ai`)  
     - Credentials: Assign Google Calendar OAuth2 credentials  
   - **Update event:**  
     - Same type as above  
     - Operation: Update event  
     - Parameters: `eventId` from AI agent override expression  
   - **Find single event:**  
     - Operation: Get event by ID  
     - Event ID: from AI agent override  
   - **Find multiple events:**  
     - Operation: Get all events  
     - Limit: 10

9. **Create Gmail Nodes:**  
   - **Draft email:**  
     - Type: `n8n-nodes-base.gmailTool`  
     - Resource: Draft  
     - Subject and Message: set via AI agent override expressions (e.g., `$fromAI('Subject')`)  
     - Credentials: Gmail OAuth2 credentials  
   - **Find emails:**  
     - Operation: Get all emails  
     - Filters: Optional, from AI agent overrides  
     - Credentials: Gmail OAuth2 credentials  

10. **Create Google Sheets Nodes for CRM:**  
    - **Add new row:**  
      - Type: `n8n-nodes-base.googleSheetsTool`  
      - Operation: Append  
      - Document ID and Sheet Name: your CRM Google Sheet (e.g., contacts list)  
      - Credentials: Google Sheets OAuth2  
    - **Find row:**  
      - Operation: Find row  
      - Same document and sheet as above  
    - **Update row:**  
      - Operation: Update row  
      - Matching column: `row_number`  
      - Same document and sheet

11. **Link MCP Server Trigger outputs to all app nodes** (Google Calendar, Gmail, Google Sheets nodes) via the `ai_tool` connections.

12. **Connect Simple Memory node as AI memory input to the Personal Assistant node.**

13. **Add Sticky Note nodes with descriptive content for user guidance (optional but recommended).**

14. **Configure all credentials properly and test webhook URLs to ensure connectivity.**

15. **Activate the workflow and test by sending chat messages to the initial webhook.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                            | Context or Link                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| This workflow demonstrates the synergy between AI natural language understanding (Google Gemini), MCP protocol orchestration, and app automation (Google Calendar, Gmail, Google Sheets). It serves as a foundational template for building conversational personal assistants.                           | Workflow purpose                                      |
| For setup, ensure you have valid API credentials for Google Gemini (Google PaLM API), Google Calendar OAuth2, Gmail OAuth2, and Google Sheets OAuth2 configured in n8n.                                                                                                                                | Credential requirements                               |
| The workflow includes detailed sticky notes describing example use cases for calendar, email, and CRM nodes to guide customization and expansion.                                                                                                                                                     | Sticky notes within workflow                           |
| To extend, consider adding more apps like Slack, Microsoft Teams, or task/project management tools via additional MCP-enabled nodes.                                                                                                                                                                  | Enhancement suggestions                               |
| For notifications on completed tasks, integrate output nodes such as Telegram or Microsoft Teams, triggered on workflow completion as per Sticky Note3.                                                                                                                                                | Notification integration note                          |
| Contact 1 Node (https://1node.ai) for consulting or assistance in building customized n8n personal assistant workflows.                                                                                                                                                                               | External contact link                                 |

---

This document should allow advanced users and AI agents to understand, reproduce, and extend the Personal Assistant MCP server workflow effectively.