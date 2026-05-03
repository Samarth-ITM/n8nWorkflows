Manage Google Calendar Events with OpenAI Assistant (Get, Create, Delete)

https://n8nworkflows.xyz/workflows/manage-google-calendar-events-with-openai-assistant--get--create--delete--7787


# Manage Google Calendar Events with OpenAI Assistant (Get, Create, Delete)

### 1. Workflow Overview

This workflow enables advanced management of Google Calendar events through an OpenAI-powered conversational assistant. It is designed as a sub-agent workflow that receives textual commands and a session ID from a parent workflow, interprets user intent via an AI agent, and performs calendar operations accordingly.

**Target Use Cases:**  
- Getting calendar events within a specified date range.  
- Creating new calendar events with start/end times and summary details.  
- Deleting existing events by event ID.

**Logical Blocks:**  

- **1.1 Input Reception:** Receives input from a parent workflow, including user text and session identifier.  
- **1.2 AI Processing:** Uses an OpenAI chat model combined with LangChain agent logic and session-based memory to understand user commands and select the appropriate calendar operation tool.  
- **1.3 Calendar Operations:** Executes Google Calendar API calls to get, create, or delete events based on AI agent instructions.  
- **1.4 Contextual Memory:** Maintains conversational context keyed by session ID to enable coherent multi-turn dialogue.  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block acts as the entry point, triggered when called by another workflow. It accepts user input text and session ID parameters to initiate processing.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Receives external workflow execution calls with specific input parameters.  
    - Configuration: Expects inputs named `text` (user command) and `sessionid` (unique session key).  
    - Inputs: Triggered externally, no upstream nodes.  
    - Outputs: Sends data to the AI Agent node.  
    - Edge Cases: Missing `text` or `sessionid` parameters could cause downstream failures; no built-in validation.  
    - Version: 1.1  

---

#### 2.2 AI Processing

- **Overview:**  
  This block interprets the input text using a LangChain AI agent integrated with an OpenAI GPT-4.1-mini chat model, leveraging session-based memory to maintain context. It determines which calendar operation to perform and formulates the necessary parameters.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**  

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Processes natural language input to select and invoke appropriate calendar tools ("Get", "Create", "Delete").  
    - Configuration:  
      - Input Text: Bound dynamically to incoming `text` JSON property.  
      - System Message: Defines assistant role as calendar manager; instructs on tool usage and required parameters (start, end, summary).  
      - Prompt Type: `define` (custom prompt).  
    - Inputs: Receives input from "When Executed by Another Workflow" node.  
    - Outputs: Sends AI-determined parameters/commands to calendar operation nodes.  
    - Edge Cases: Failure to parse ambiguous or incomplete user input; possible timeout or API errors when invoking OpenAI; missing required fields for calendar operations.  
    - Version: 2.2  
    - Sub-Workflow: This node is the core AI decision maker within the sub-agent.  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4.1-mini chat completions to the AI Agent for natural language understanding and generation.  
    - Configuration:  
      - Model: GPT-4.1-mini selected for lightweight, capable inference.  
      - Options: Default.  
    - Inputs: Receives prompts from AI Agent node.  
    - Outputs: Sends responses back to AI Agent.  
    - Credentials: Requires valid OpenAI API key configured (`OpenAi account`).  
    - Edge Cases: Authentication failures, rate limits, request timeouts.  
    - Version: 1.2  

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Stores conversation history keyed by `sessionid` to maintain multi-turn context.  
    - Configuration:  
      - Session Key: Dynamically bound to incoming `sessionid` JSON property.  
      - Session ID Type: Custom key.  
    - Inputs: Connected as AI memory input to AI Agent.  
    - Outputs: Feeds memory context to AI Agent node.  
    - Edge Cases: Missing or invalid session IDs could break context; memory overflow if session history grows too large.  
    - Version: 1.3  

---

#### 2.3 Calendar Operations

- **Overview:**  
  This block contains three Google Calendar API tool nodes that execute calendar event retrieval, creation, or deletion as directed by the AI Agent.

- **Nodes Involved:**  
  - Get  
  - Create  
  - Delete

- **Node Details:**  

  - **Get**  
    - Type: Google Calendar Tool - Get All Events  
    - Role: Retrieves calendar events within a date range specified by AI Agent.  
    - Configuration:  
      - Operation: `getAll`  
      - Calendar: Set to fixed account `test@gmail.com`.  
      - Parameters:  
        - `timeMin` and `timeMax` dynamically bound from AI Agent parameters 'After' and 'Before'.  
        - `returnAll` bound to AI Agent parameter `Return_All` boolean.  
    - Inputs: Receives parameters from AI Agent via `ai_tool` connection.  
    - Outputs: Event list data returned to AI Agent.  
    - Credentials: Uses OAuth2 Google Calendar credential named `Google Calendar account`.  
    - Edge Cases: Invalid date range, authentication failure, API rate limits.  
    - Version: 1.3  

  - **Create**  
    - Type: Google Calendar Tool - Create Event  
    - Role: Creates new calendar events with required parameters.  
    - Configuration:  
      - Calendar: Fixed to `test@gmail.com`.  
      - Parameters:  
        - `start`, `end` dates from AI Agent parameters.  
        - `summary` from AI Agent.  
        - `useDefaultReminders` boolean optionally set.  
    - Inputs: AI Agent provides event details.  
    - Outputs: Confirmation of created event passed back to AI Agent.  
    - Credentials: OAuth2 Google Calendar account.  
    - Edge Cases: Missing or invalid date/time, missing summary, auth errors.  
    - Version: 1.3  

  - **Delete**  
    - Type: Google Calendar Tool - Delete Event  
    - Role: Deletes an event by its event ID.  
    - Configuration:  
      - Calendar: Fixed `test@gmail.com`.  
      - Parameters:  
        - `eventId` dynamically provided by AI Agent.  
      - Operation: `delete`  
    - Inputs: AI Agent passes event ID for deletion.  
    - Outputs: Deletion confirmation returned to AI Agent.  
    - Credentials: OAuth2 Google Calendar account.  
    - Edge Cases: Invalid or missing event ID, event not found, auth failure.  
    - Version: 1.3  

---

#### 2.4 Contextual Notes

- **Sticky Note** (position [-592,-16])  
  - Describes the calendar tool nodes collectively as a "CHILD — CALENDAR TOOLS" block.  
  - Highlights:  
    - Get: lists events by date range  
    - Create: requires start, end, summary  
    - Delete: requires eventId  
    - Memory keyed by `sessionid`  

- **Sticky Note1** (position [-592,144])  
  - Describes overall sub-agent logic:  
    - Inputs: text, sessionid  
    - AI Agent selects the appropriate tool  
    - Returns concise confirmation to parent workflow  

---

### 3. Summary Table

| Node Name                    | Node Type                                | Functional Role                     | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                  |
|------------------------------|-----------------------------------------|-----------------------------------|--------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger                | Entry point; receives `text` and `sessionid` | (external trigger)             | AI Agent                      |                                                                                        |
| AI Agent                     | LangChain Agent                         | Interpret input, select calendar tool to invoke | When Executed by Another Workflow | Get, Create, Delete           | Sub-agent logic: inputs text & sessionid, selects tool, returns confirmation               |
| OpenAI Chat Model            | LangChain OpenAI Chat Model             | Provides GPT-4.1-mini completions | AI Agent                       | AI Agent                      |                                                                                        |
| Simple Memory                | LangChain Memory Buffer Window          | Maintains conversation context keyed by `sessionid` | AI Agent                       | AI Agent                      | Memory keyed by `sessionid`                                                               |
| Get                         | Google Calendar Tool                    | Retrieve events by date range     | AI Agent                      | AI Agent                      | CHILD — CALENDAR TOOLS: Get lists events by date range                                   |
| Create                      | Google Calendar Tool                    | Create new calendar event         | AI Agent                      | AI Agent                      | CHILD — CALENDAR TOOLS: Create requires start, end, summary                              |
| Delete                      | Google Calendar Tool                    | Delete event by event ID          | AI Agent                      | AI Agent                      | CHILD — CALENDAR TOOLS: Delete requires eventId                                          |
| Sticky Note                 | Sticky Note                            | Describes calendar tool nodes     |                                |                               | CHILD — CALENDAR TOOLS: Details about Get, Create, Delete, and memory usage               |
| Sticky Note1                | Sticky Note                            | Describes sub-agent logic          |                                |                               | SUB-AGENT LOGIC: Inputs and tool selection, confirmation return                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add node: **Execute Workflow Trigger** (When Executed by Another Workflow)  
   - Configure parameters:  
     - Workflow Inputs: Add two fields: `text` (string), `sessionid` (string)  
   - Position accordingly.  

2. **Add AI Agent Node**  
   - Add node: **LangChain Agent**  
   - Set input text expression: `={{ $json.text }}`  
   - Define system message prompt:  
     ```
     You are a calendar assistant who can easily manage events in a google calendar. 
     You must provide the start and end date.

     If you need to create events in the calendar use the "Get" tool. To delete events in the calendar use "Delete", and to create events use the "Create" tool.

     the current date and time is {{ $now}}
     Additionally, you must provide the summary and other details for the calendar which your tool might need.
     ```  
   - Set promptType to `define`  
   - Connect **Execute Workflow Trigger** main output to this node's main input.  

3. **Configure OpenAI Chat Model Node**  
   - Add node: **LangChain OpenAI Chat Model**  
   - Select model: `gpt-4.1-mini`  
   - Link this node as `ai_languageModel` input to AI Agent node.  
   - Assign OpenAI API credentials (create or select existing with valid API key).  

4. **Configure Simple Memory Node**  
   - Add node: **LangChain Memory Buffer Window**  
   - Set sessionKey expression: `={{ $json.sessionid }}`  
   - Set sessionIdType to `customKey`  
   - Connect this node as `ai_memory` input to AI Agent node.  

5. **Add Google Calendar 'Get' Node**  
   - Add node: **Google Calendar Tool**  
   - Set operation to `getAll`  
   - Set calendar to fixed email (e.g., `test@gmail.com`)  
   - Set parameters:  
     - `timeMin` = `={{ $fromAI('After', '', 'string') }}`  
     - `timeMax` = `={{ $fromAI('Before', '', 'string') }}`  
     - `returnAll` = `={{ $fromAI('Return_All', '', 'boolean') }}`  
   - Connect AI Agent's `ai_tool` output to this node's input for `Get` operation.  
   - Assign Google Calendar OAuth2 credentials.  

6. **Add Google Calendar 'Create' Node**  
   - Add node: **Google Calendar Tool**  
   - Set operation to `create` (default)  
   - Set calendar to fixed email (`test@gmail.com`)  
   - Set parameters:  
     - `start` = `={{ $fromAI('Start', '', 'string') }}`  
     - `end` = `={{ $fromAI('End', '', 'string') }}`  
     - `summary` = `={{ $fromAI('Summary', '', 'string') }}`  
     - `useDefaultReminders` = `={{ $fromAI('Use_Default_Reminders', '', 'boolean') }}` (optional)  
   - Connect AI Agent's `ai_tool` output to this node for `Create` operation.  
   - Assign Google Calendar OAuth2 credentials.  

7. **Add Google Calendar 'Delete' Node**  
   - Add node: **Google Calendar Tool**  
   - Set operation to `delete`  
   - Set calendar to fixed email (`test@gmail.com`)  
   - Set parameter:  
     - `eventId` = `={{ $fromAI('Event_ID', '', 'string') }}`  
   - Connect AI Agent's `ai_tool` output to this node for `Delete` operation.  
   - Assign Google Calendar OAuth2 credentials.  

8. **Add Sticky Notes** (optional, for clarity)  
   - Add one sticky note near calendar tools describing their roles and required parameters.  
   - Add one sticky note near AI Agent describing sub-agent logic and inputs.  

9. **Verify Connections and Credentials**  
   - Ensure all nodes are properly connected and credential IDs are correctly assigned.  
   - Test execution with sample input JSON:  
     ```json
     {
       "text": "Show me all events next week",
       "sessionid": "user-session-123"
     }
     ```  
   - Observe AI Agent selecting the "Get" tool and returning event data.  

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow acts as a sub-agent designed to be called by a parent workflow, facilitating modular AI conversational calendar management. | Internal design philosophy for modular agent workflows.                                         |
| The AI Agent’s system message explicitly instructs tool usage and required parameters, crucial for correct operation.                   | System prompt text inside AI Agent node configuration.                                          |
| OpenAI GPT-4.1-mini was chosen to balance capability and speed for chat completions.                                           | Model selection rationale.                                                                       |
| Google Calendar OAuth2 credentials must be pre-configured with appropriate scopes (read/write calendar) for tool nodes to work.          | Credential setup requirement.                                                                    |
| Memory node uses `sessionid` to maintain conversation context per user/session, enabling multi-turn dialogue.                           | Conversation state management strategy.                                                         |
| Sticky notes provide helpful in-editor documentation for maintainers and collaborators.                                    | n8n UI documentation practice.                                                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.