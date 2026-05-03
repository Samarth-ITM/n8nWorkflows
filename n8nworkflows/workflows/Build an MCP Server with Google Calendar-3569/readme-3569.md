Build an MCP Server with Google Calendar

https://n8nworkflows.xyz/workflows/build-an-mcp-server-with-google-calendar-3569


# Build an MCP Server with Google Calendar

### 1. Workflow Overview

This workflow, titled **"Build an MCP Server with Google Calendar"**, is designed to integrate the MCP (Multi-Channel Processing) framework with Google Calendar within n8n. It targets developers, data analysts, and automation enthusiasts who want to automate synchronization between Google Calendar events and AI Agents using MCP.

The workflow is logically divided into two main parts:

- **1.1 MCP Server Setup with Google Calendar Tools:**  
  This block sets up an MCP Server in n8n that exposes Google Calendar operations (search, create, update, delete events) as MCP tools. It includes the MCP Server trigger and Google Calendar nodes configured with OAuth2 credentials.

- **1.2 MCP Client and AI Agent Integration:**  
  This block creates an AI Agent workflow that listens for chat messages, uses a language model (LLM), maintains conversation memory, and connects to the MCP Server via an MCP Client node. This enables the AI Agent to detect and react to changes in Google Calendar events automatically.

Supporting the main logic are multiple sticky notes that provide detailed instructions, setup guidance, and contextual information for users.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Setup with Google Calendar Tools

**Overview:**  
This block configures the MCP Server trigger and integrates Google Calendar tools (search, create, update, delete events) as MCP tools. It enables the MCP Server to expose Google Calendar operations for consumption by MCP Clients.

**Nodes Involved:**  
- Google Calendar MCP (MCP Server Trigger)  
- SearchEvent (Google Calendar Tool)  
- CreateEvent (Google Calendar Tool)  
- UpdateEvent (Google Calendar Tool)  
- DeleteEvent (Google Calendar Tool)  
- Sticky Notes: Sticky Note2, Sticky Note4, Sticky Note5, Sticky Note6, Sticky Note8, Sticky Note9, Sticky Note12, Sticky Note13

**Node Details:**

- **Google Calendar MCP**  
  - Type: MCP Server Trigger (LangChain MCP Trigger)  
  - Role: Entry point for MCP Server events; listens for incoming MCP client requests.  
  - Configuration: Webhook path set to `/my-calendar`.  
  - Input: None (trigger node).  
  - Output: Connected as `ai_tool` to all Google Calendar tool nodes.  
  - Edge Cases: Webhook URL must be publicly accessible; activation required for listening.  
  - Notes: URL copied later for MCP Client use.

- **SearchEvent**  
  - Type: Google Calendar Tool  
  - Role: Retrieves multiple calendar events based on time range and limit parameters.  
  - Configuration: Uses OAuth2 credentials for Google Calendar; parameters like `timeMin`, `timeMax`, and `limit` are dynamically set via AI overrides.  
  - Input: Receives MCP Server trigger calls.  
  - Output: Returns event list to MCP Server trigger.  
  - Edge Cases: API quota limits, invalid date formats, auth token expiration.

- **CreateEvent**  
  - Type: Google Calendar Tool  
  - Role: Creates a new calendar event with specified start/end times, summary, and description.  
  - Configuration: OAuth2 credentials; event details dynamically populated from AI inputs.  
  - Input: MCP Server trigger calls.  
  - Output: Confirmation of event creation.  
  - Edge Cases: Invalid event data, permission errors.

- **UpdateEvent**  
  - Type: Google Calendar Tool  
  - Role: Updates an existing event identified by event ID with new details.  
  - Configuration: OAuth2 credentials; event ID and fields dynamically set.  
  - Input: MCP Server trigger calls.  
  - Output: Confirmation of event update.  
  - Edge Cases: Event not found, invalid event ID, permission issues.

- **DeleteEvent**  
  - Type: Google Calendar Tool  
  - Role: Deletes an event by event ID.  
  - Configuration: OAuth2 credentials; event ID dynamically set.  
  - Input: MCP Server trigger calls.  
  - Output: Confirmation of deletion.  
  - Edge Cases: Event not found, permission errors.

- **Sticky Notes**  
  - Provide step-by-step instructions, setup tips, author info, and motivational content.  
  - Some notes include links to credential setup videos and visual aids for activation steps.

---

#### 2.2 MCP Client and AI Agent Integration

**Overview:**  
This block builds an AI Agent workflow that triggers on chat messages, uses a language model (LLM) to process input, maintains conversation memory, and connects to the MCP Server via an MCP Client tool. This setup allows the AI Agent to detect and respond to Google Calendar changes in real time.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- AI Agent (LangChain Agent)  
- Simple Memory (Memory Buffer Window)  
- Calendar MCP (MCP Client Tool)  
- gpt-4o (OpenAI Chat Model)  
- Sticky Notes: Sticky Note7, Sticky Note10, Sticky Note11

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger (LangChain chatTrigger)  
  - Role: Starts the AI Agent workflow upon receiving a chat message.  
  - Configuration: Webhook with unique ID; no additional parameters.  
  - Input: External chat messages.  
  - Output: Triggers AI Agent node.  
  - Edge Cases: Webhook availability, message format validation.

- **AI Agent**  
  - Type: LangChain Agent node  
  - Role: Processes chat messages using an LLM and integrates MCP Client tool.  
  - Configuration:  
    - System message preset: "You are a helpful assistant. Current datetime is {{ $now.toString() }}"  
    - Connected to LLM node (`gpt-4o`) for language processing.  
    - Connected to Simple Memory for conversation context.  
    - Connected to MCP Client tool (`Calendar MCP`) for Google Calendar data.  
  - Input: Chat messages from trigger.  
  - Output: Responses to chat messages.  
  - Edge Cases: Model API limits, memory overflow, expression evaluation errors.

- **Simple Memory**  
  - Type: Memory Buffer Window (LangChain memory)  
  - Role: Maintains conversational context for the AI Agent.  
  - Configuration: Default buffer window memory.  
  - Input: AI Agent node.  
  - Output: AI Agent node.  
  - Edge Cases: Memory size limits, data persistence.

- **Calendar MCP**  
  - Type: MCP Client Tool (LangChain mcpClientTool)  
  - Role: Connects to MCP Server to receive real-time Google Calendar event updates.  
  - Configuration: SSE Endpoint URL set to MCP Server URL copied from `Google Calendar MCP` node (e.g., `https://xxx.app.n8n.cloud/mcp/my-calendar/sse`).  
  - Input: AI Agent node (as a tool).  
  - Output: AI Agent node.  
  - Edge Cases: SSE connection failures, URL misconfiguration, network issues.

- **gpt-4o**  
  - Type: OpenAI Chat Model (LangChain lmChatOpenAi)  
  - Role: Provides the language model for AI Agent responses.  
  - Configuration: Model set to `gpt-4o-mini` (noted that full `gpt-4o` performs better).  
  - Credentials: OpenAI API key configured.  
  - Input: AI Agent node.  
  - Output: AI Agent node.  
  - Edge Cases: API rate limits, model availability.

- **Sticky Notes**  
  - Explain how to create the AI Agent workflow, add memory, select LLM, and connect MCP Client.  
  - Provide rationale for model choice and encouragement.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                          | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                   |
|---------------------------|----------------------------------|----------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note8              | Sticky Note                      | General notes                          |                             |                             |                                                                                              |
| Sticky Note9              | Sticky Note                      | Title and introduction                 |                             |                             | # Learn How to Build a MCP Server with Google Calendar                                      |
| Sticky Note2              | Sticky Note                      | Introduction to MCP with Google Calendar |                             |                             | # Introduce tutorial and MCP Server features                                                |
| Sticky Note5              | Sticky Note                      | Author information                     |                             |                             | Author: SunGuannan, freelance consultant, contact: sguann2023@gmail.com                      |
| Sticky Note4              | Sticky Note                      | Google Credentials setup guidance     |                             |                             | Video link: https://www.youtube.com/watch?v=3Ai1EPznlAc                                    |
| Sticky Note              | Sticky Note                      | Step 2: MCP Server Trigger setup      |                             |                             | Instructions to add MCP Server Trigger node                                                 |
| Google Calendar MCP       | MCP Server Trigger               | MCP Server entry point                 |                             | SearchEvent, CreateEvent, UpdateEvent, DeleteEvent |                                                                                              |
| SearchEvent               | Google Calendar Tool             | Retrieve multiple calendar events     | Google Calendar MCP          | Google Calendar MCP          |                                                                                              |
| CreateEvent               | Google Calendar Tool             | Create new calendar event              | Google Calendar MCP          | Google Calendar MCP          |                                                                                              |
| UpdateEvent               | Google Calendar Tool             | Update existing calendar event        | Google Calendar MCP          | Google Calendar MCP          |                                                                                              |
| DeleteEvent               | Google Calendar Tool             | Delete calendar event                  | Google Calendar MCP          | Google Calendar MCP          |                                                                                              |
| Sticky Note6              | Sticky Note                      | Step 3: Add Google Calendar tools     |                             |                             | Instructions for adding Google Calendar tools                                               |
| Sticky Note1              | Sticky Note                      | Step 4: Copy MCP Server URL and activate |                             |                             | Instructions on copying URL and activating workflow                                         |
| Sticky Note7              | Sticky Note                      | Step 5: Create AI Agent workflow      |                             |                             | Instructions to add Chat Message trigger node                                              |
| When chat message received| Chat Trigger                    | Trigger AI Agent on chat message      |                             | AI Agent                    |                                                                                              |
| AI Agent                  | LangChain Agent                  | Process chat with LLM and MCP Client  | When chat message received, gpt-4o, Simple Memory, Calendar MCP |                             | System message includes current datetime                                                   |
| Simple Memory             | Memory Buffer Window             | Maintain conversation context         | AI Agent                    | AI Agent                    |                                                                                              |
| Calendar MCP              | MCP Client Tool                 | Connect to MCP Server for calendar data | AI Agent                    | AI Agent                    | SSE Endpoint URL copied from MCP Server node                                               |
| gpt-4o                    | OpenAI Chat Model               | Language model for AI Agent            | AI Agent                    | AI Agent                    | Model set to gpt-4o-mini; note on model performance                                         |
| Sticky Note11             | Sticky Note                      | Model choice explanation               |                             |                             | Explanation why gpt-4o preferred over gpt-4o-mini                                           |
| Sticky Note12             | Sticky Note                      | Encouragement and visual aids          |                             |                             | Visuals for creating and finishing events                                                  |
| Sticky Note13             | Sticky Note                      | Closing encouragement                  |                             |                             | # Enjoy It! 😊 😊 😊                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add MCP Server Trigger node:**  
   - Search for `MCP Server Trigger` (LangChain MCP Trigger).  
   - Set the webhook path to `my-calendar`.  
   - Save and activate the workflow after setup.

3. **Add Google Calendar Tool nodes:**  
   - Add four Google Calendar Tool nodes: `SearchEvent`, `CreateEvent`, `UpdateEvent`, `DeleteEvent`.  
   - Configure each node with your Google Calendar OAuth2 credentials.  
   - For each node, set the calendar to your Google account email (e.g., `sguann2023@gmail.com`).  
   - Configure parameters:  
     - `SearchEvent`: operation `getAll`, dynamic `timeMin`, `timeMax`, and `limit` via expressions.  
     - `CreateEvent`: set `start`, `end`, `summary`, `description` dynamically.  
     - `UpdateEvent`: operation `update`, set `eventId`, and update fields dynamically.  
     - `DeleteEvent`: operation `delete`, set `eventId` dynamically.

4. **Connect all Google Calendar Tool nodes as outputs of the MCP Server Trigger node** using the `ai_tool` connection type.

5. **Copy the MCP Server Trigger webhook URL:**  
   - Open the MCP Server Trigger node details.  
   - Copy the production URL (e.g., `https://xxx/mcp/my-calendar/sse`).  
   - Activate the workflow to start listening.

6. **Create a new workflow for the AI Agent:**  
   - Add a `Chat Trigger` node (`When chat message received`).  
   - Add an `AI Agent` node and connect it to the Chat Trigger node.  
   - In the AI Agent node:  
     - Set the system message to:  
       ```
       You are a helpful assistant.
       Current datetime is {{ $now.toString() }}
       ```  
     - Add memory by connecting a `Memory Buffer Window` node (`Simple Memory`) to the AI Agent node.  
     - Add an MCP Client Tool node (`Calendar MCP`) and configure its SSE Endpoint with the MCP Server URL copied earlier.  
     - Connect the MCP Client Tool node as an `ai_tool` input to the AI Agent node.

7. **Add an OpenAI Chat Model node (`gpt-4o`):**  
   - Configure it with your OpenAI API credentials.  
   - Set the model to `gpt-4o-mini` (or `gpt-4o` if available and tested).  
   - Connect it as the `ai_languageModel` input to the AI Agent node.

8. **Activate the AI Agent workflow** to start processing chat messages and interacting with the MCP Server.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Video tutorial for setting up Google Credentials in n8n.                                            | https://www.youtube.com/watch?v=3Ai1EPznlAc                                                        |
| Author: SunGuannan, freelance consultant specializing in automations and data analysis.             | Contact: sguann2023@gmail.com                                                                       |
| Explanation on why the `gpt-4o` model is preferred over `gpt-4o-mini` for handling calendar requests.| Sticky Note11                                                                                       |
| Visual aids for creating and finishing calendar events in the workflow.                             | Sticky Note12                                                                                       |
| Encouragement and positive closing message for users.                                               | Sticky Note13                                                                                       |

---

This documentation provides a complete, structured understanding of the workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the integration of MCP with Google Calendar in n8n.