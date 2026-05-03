Manage Google Calendar events with GPT‑4 and an AI assistant

https://n8nworkflows.xyz/workflows/manage-google-calendar-events-with-gpt-4-and-an-ai-assistant-13734


# Manage Google Calendar events with GPT‑4 and an AI assistant

# Workflow Reference: Manage Google Calendar events with GPT‑4 and an AI assistant

## 1. Workflow Overview
This workflow functions as an intelligent intermediary (sub-workflow) that allows users to manage their Google Calendar using natural language. By utilizing an AI Agent powered by GPT-4, the system interprets unstructured text commands—such as "Schedule a coffee chat tomorrow at 10 AM"— and executes the corresponding technical actions in Google Calendar.

The logic is organized into four functional blocks:
- **1.1 Input Reception:** Receives the natural language query from a parent workflow.
- **1.2 AI Processing:** Uses a Chat Model and a System Prompt to determine which calendar operation to perform.
- **1.3 Tool Execution:** A suite of Google Calendar tools that the AI calls to Create, Read, Update, or Delete events.
- **1.4 Response Handling:** Captures the outcome of the AI's action and returns a success or error message to the requester.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception
**Overview:** This block acts as the entry point, accepting data from external triggers or parent workflows.
- **Nodes Involved:** `Receive Calendar Query`.
- **Node Details:**
    - **Type:** Execute Workflow Trigger.
    - **Configuration:** Defines a required JSON input schema containing a single key: `query` (String).
    - **Connections:** Passes the query directly to the AI Agent.

### 2.2 AI Processing
**Overview:** The "brain" of the workflow. It interprets the user's intent and manages the flow between tools.
- **Nodes Involved:** `Manage Calendar with AI Agent`, `OpenAI GPT-4.1 Chat Model`.
- **Node Details:**
    - **Manage Calendar with AI Agent:**
        - **Type:** AI Agent (LangChain).
        - **Configuration:** Uses a detailed System Message defining the "Calendar Assistant" profile. It includes instructions on default durations (1 hour) and safety rules (must "Get Events" before updating or deleting).
        - **Key Expressions:** Uses `{{ $json.query }}` as the input prompt and `{{ $now }}` to provide real-time context to the AI.
        - **Error Handling:** Set to "Continue (Error Output)" to route failures to the Error Response block.
    - **OpenAI GPT-4.1 Chat Model:**
        - **Type:** Chat OpenAI Language Model.
        - **Configuration:** Connects to the `gpt-4.1` model.
        - **Role:** Provides the reasoning capabilities for the Agent.

### 2.3 Tool Execution (Google Calendar Tools)
**Overview:** A collection of specialized functions available to the AI Agent to interact with the Google Calendar API.
- **Nodes Involved:** `Get Calendar Events`, `Create Calendar Event`, `Create Calendar Event with Attendee`, `Update Calendar Event`, `Delete Calendar Event`.
- **Node Details:**
    - **Common Configuration:** All nodes require Google Calendar OAuth2 credentials and a Calendar ID (defaulted in the template to `user@example.com`).
    - **Data Extraction:** These nodes use the `$fromAI()` function to extract parameters identified by the GPT model (e.g., `eventStart`, `eventEnd`, `eventTitle`, `eventID`).
    - **Specific Logic:** 
        - `Get Calendar Events`: Uses `timeMin` and `timeMax` to filter events around a specific date.
        - `Create Event with Attendee`: Specifically maps the `eventAttendeeEmail` field to the attendees list.

### 2.4 Response Handling
**Overview:** Formats the final output to ensure the parent workflow receives a clean response.
- **Nodes Involved:** `Set Success Response`, `Set Error Response`.
- **Node Details:**
    - **Set Success Response:** Captures the AI Agent's text output (`{{$json.output}}`) and maps it to a standard `response` variable.
    - **Set Error Response:** Provides a hardcoded fallback message: "Unable to perform task. Please try again." in case the AI Agent node fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Calendar Query | Execute Workflow Trigger | Input Entry Point | None | AI Agent | 1. Receive Query: Sub-workflow trigger accepts a `query` string. |
| Manage Calendar with AI Agent | AI Agent | Logic Controller | Receive Query, Tools | Response Nodes | 2. AI Calendar Agent: GPT-4.1 interprets the query and executes tools. |
| OpenAI GPT-4.1 Chat Model | Chat OpenAI | AI Engine | None | AI Agent | 2. AI Calendar Agent: GPT-4.1 interprets the query and executes tools. |
| Get Calendar Events | Google Calendar Tool | Data Retrieval | AI Agent | AI Agent | 3. Google Calendar Tools: Agent must retrieve events before updating. |
| Update Calendar Event | Google Calendar Tool | Data Modification | AI Agent | AI Agent | 3. Google Calendar Tools: Five calendar operations available. |
| Delete Calendar Event | Google Calendar Tool | Data Removal | AI Agent | AI Agent | 3. Google Calendar Tools: Five calendar operations available. |
| Create Calendar Event | Google Calendar Tool | Data Creation | AI Agent | AI Agent | 3. Google Calendar Tools: Five calendar operations available. |
| Create Calendar Event with Attendee | Google Calendar Tool | Data Creation (Invite) | AI Agent | AI Agent | 3. Google Calendar Tools: Five calendar operations available. |
| Set Success Response | Set | Output Formatting | AI Agent | None | 4. Return Response: Success returns the AI agent's output. |
| Set Error Response | Set | Error Formatting | AI Agent | None | 4. Return Response: Error returns a fallback message. |

---

## 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create an **Execute Workflow Trigger** node. Define a JSON example with a `query` field.
2.  **AI Agent Setup:** 
    - Add an **AI Agent** node. Set the prompt type to "Define" and input `{{ $json.query }}`.
    - Paste the System Message defining the tools (Create, Get, Update, Delete) and the rule to always "Get" before "Update/Delete".
3.  **Model Connection:** Attach an **OpenAI Chat Model** node to the AI Agent. Select `gpt-4` or higher and provide your OpenAI credentials.
4.  **Calendar Tools Integration:** Create five **Google Calendar** nodes and connect them to the **Tool** input of the AI Agent:
    - **Get Events:** Set operation to "Get All". Use expressions like `{{ $fromAI('Before') }}` for time limits.
    - **Create Event:** Set operation to "Create". Map start/end times using `{{ $fromAI('eventStart') }}`.
    - **Create with Attendee:** Same as Create, but include the "Attendees" field mapped to `{{ $fromAI('eventAttendeeEmail') }}`.
    - **Update Event:** Set operation to "Update". Map `Event ID` to `{{ $fromAI('eventID') }}`.
    - **Delete Event:** Set operation to "Delete". Map `Event ID` to `{{ $fromAI('eventID') }}`.
5.  **Credential Setup:** Ensure all Google Calendar nodes use a valid **Google Calendar OAuth2** credential. Update the "Calendar ID" in each node to your email address or `primary`.
6.  **Output Routing:**
    - Connect the AI Agent's main output to a **Set** node named "Set Success Response". Map `response` to `{{ $json.output }}`.
    - Enable "On Error -> Continue" on the AI Agent node.
    - Connect the second (error) output of the AI Agent to a **Set** node named "Set Error Response" with a static fallback message.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Contact & Feedback** | [Start the conversation here](https://tally.so/r/EkKGgB) |
| **Project Attribution** | Developed by Milo Bravo | BRaiA Labs |
| **Professional Profile** | [Milo Bravo LinkedIn](https://linkedin.com/in/MiloBravo/) |
| **Setup Requirement** | Must update Calendar ID from `user@example.com` to your actual Gmail address. |