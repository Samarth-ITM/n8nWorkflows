Manage Google Calendar Events with Natural Language Using Gemini 1.5 Flash

https://n8nworkflows.xyz/workflows/manage-google-calendar-events-with-natural-language-using-gemini-1-5-flash-5165


# Manage Google Calendar Events with Natural Language Using Gemini 1.5 Flash

### 1. Workflow Overview

This workflow enables managing Google Calendar events via natural language queries using Google’s Gemini 1.5 Flash AI model. It targets users who want to create or retrieve calendar events by chatting naturally, without manual calendar interaction.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives user chat messages as natural language input.
- **1.2 AI Processing:** Uses the Gemini 1.5 Flash model with memory to interpret user intent, extract event details (e.g., start/end times, summary), and decide whether to add or get events.
- **1.3 Calendar Interaction:** Invokes Google Calendar nodes to either fetch events in a date range or add a new event based on AI-extracted data.
- **1.4 Conversation Memory:** Stores conversation context to enable continuity in chat sessions.
- **1.5 User Guidance (Sticky Notes):** Provides inline documentation to clarify node purposes and workflow logic.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures chat messages from users to initiate the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type:* Chat trigger (LangChain chatTrigger node)  
    - *Config:* Listens for incoming chat messages via webhook (webhookId provided)  
    - *Input:* External chat messages from user  
    - *Output:* Passes message to AI Agent node  
    - *Failure cases:* Webhook failure, message format errors, network issues  
    - *Sticky Note:* “Send the prompt if we want to set the event on the calendar or get events available in the calendar”

#### 1.2 AI Processing

- **Overview:**  
  Processes user input with an AI agent that uses Google Gemini 1.5 Flash, enriched with conversation memory, to extract event-related information and system instructions.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Simple Memory

- **Node Details:**  
  - **AI Agent**  
    - *Type:* LangChain agent node  
    - *Config:* Contains a system message setting the assistant role, current date, and Google Calendar tool usage instructions. It expects variables `afterDate` and `beforeDate` in ISO format to fetch events.  
    - *Input:* Receives chat messages from “When chat message received” node  
    - *Output:* Sends extracted parameters and commands to calendar nodes  
    - *Failure cases:* Parsing errors, unexpected input, AI service downtime  
  - **Google Gemini Chat Model**  
    - *Type:* LangChain language model node (Google Gemini 1.5 Flash)  
    - *Config:* Uses modelName `models/gemini-1.5-flash` for natural language understanding  
    - *Input:* Connected as AI language model for AI Agent  
    - *Output:* AI Agent receives model completions  
    - *Failure cases:* Model API errors, request timeouts  
  - **Simple Memory**  
    - *Type:* LangChain memoryBufferWindow node  
    - *Config:* Stores conversation history to maintain context over multiple interactions  
    - *Input:* AI Agent outputs conversation text to memory  
    - *Output:* Passes memory context to AI Agent on subsequent runs  
    - *Failure cases:* Memory overflow, data corruption  
    - *Sticky Note:* “Inbuilt memory for n8n which will store the value prompt received by AI agent so model can remember the conversation”

#### 1.3 Calendar Interaction

- **Overview:**  
  Performs Google Calendar operations based on AI instructions — either querying events within a timeframe or adding a new event.

- **Nodes Involved:**  
  - Google Calendar (Add Event)  
  - Google Calendar1 (Get Events)

- **Node Details:**  
  - **Google Calendar (Add Event)**  
    - *Type:* Google Calendar Tool node  
    - *Config:* Adds events using AI-extracted fields: `startTime`, `endTime`, and `summary`. The calendar ID is set to a specific Google group calendar.  
    - *Input:* Receives structured event data from AI Agent outputs  
    - *Output:* Confirms event creation or passes errors  
    - *Failure cases:* Authentication errors, invalid dates, API quota limits  
    - *Sticky Note:* “Calendar node responsible for adding event in the calendar”  
  - **Google Calendar1 (Get Events)**  
    - *Type:* Google Calendar Tool node  
    - *Config:* Retrieves all events between `afterDate` and `beforeDate` variables extracted by AI. Uses the same calendar ID as above.  
    - *Input:* Uses date range filters provided by AI Agent  
    - *Output:* Returns event list to AI Agent or user interface  
    - *Failure cases:* Invalid date range, no events found, API errors  
    - *Sticky Note:* “Calendar node responsible for getting all the events in asked timeframe”

#### 1.4 Conversation Memory

- **Overview:**  
  Maintains chat continuity by storing and recalling previous conversation prompts and AI output.

- **Nodes Involved:**  
  - Simple Memory (same as above)

- **Node Details:**  
  Same as described under 1.2.

#### 1.5 User Guidance (Sticky Notes)

- **Overview:**  
  Multiple sticky notes provide inline explanation of workflow logic and node functions for maintainers.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5

- **Node Details:**  
  - Sticky Note: “Send the prompt if we want to set the event on the calendar or get events available in the calendar” (covers input trigger)  
  - Sticky Note1: “Responsible for injecting the system prompt so things that are related for the project is only carried out. Also connects Gemini, Memory and calendar application together” (covers AI Agent)  
  - Sticky Note2: “Takes in the prompt and gives out the small event name based on the prompt” (covers Gemini Chat Model)  
  - Sticky Note3: “Inbuilt memory for n8n which will store the value prompt received by AI agent so model can remember the conversation” (covers Simple Memory)  
  - Sticky Note4: “Calendar node responsible for adding event in the calendar” (covers Google Calendar add event node)  
  - Sticky Note5: “Calendar node responsible for getting all the events in asked timeframe” (covers Google Calendar get events node)

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                                     | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                         |
|-------------------------|-----------------------------------------|----------------------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received | LangChain chatTrigger                   | Receives user chat messages                         | External webhook             | AI Agent                   | Send the prompt if we want to set the event on the calendar or get events available in the calendar |
| AI Agent                | LangChain agent                         | Processes input, extracts event info, orchestrates | When chat message received   | Google Calendar, Google Calendar1 | Responsible for injecting the system prompt so things that are related for the project is only carried out. Also connects Gemini, Memory and calendar application together |
| Google Gemini Chat Model | LangChain language model (Google Gemini) | Understands natural language input                  | AI Agent (ai_languageModel)  | AI Agent                   | Takes in the prompt and gives out the small event name based on the prompt                        |
| Simple Memory           | LangChain memoryBufferWindow            | Stores conversation context                         | AI Agent (ai_memory)         | AI Agent                   | Inbuilt memory for n8n which will store the value prompt received by AI agent so model can remember the conversation |
| Google Calendar         | Google Calendar Tool                    | Adds new calendar events                            | AI Agent (ai_tool)           | AI Agent                   | Calendar node responsible for adding event in the calendar                                      |
| Google Calendar1        | Google Calendar Tool                    | Retrieves calendar events in date range            | AI Agent (ai_tool)           | AI Agent                   | Calendar node responsible for getting all the events in asked timeframe                         |
| Sticky Note             | Sticky Note                            | Documentation                                       |                             |                            | Send the prompt if we want to set the event on the calendar or get events available in the calendar |
| Sticky Note1            | Sticky Note                            | Documentation                                       |                             |                            | Responsible for injecting the system prompt so things that are related for the project is only carried out. Also connects Gemini, Memory and calendar application together |
| Sticky Note2            | Sticky Note                            | Documentation                                       |                             |                            | Takes in the prompt and gives out the small event name based on the prompt                       |
| Sticky Note3            | Sticky Note                            | Documentation                                       |                             |                            | Inbuilt memory for n8n which will store the value prompt received by AI agent so model can remember the conversation |
| Sticky Note4            | Sticky Note                            | Documentation                                       |                             |                            | Calendar node responsible for adding event in the calendar                                     |
| Sticky Note5            | Sticky Note                            | Documentation                                       |                             |                            | Calendar node responsible for getting all the events in asked timeframe                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node:**  
   - Add a LangChain chatTrigger node named `When chat message received`.  
   - Configure webhookId or set up a webhook to listen for incoming chat messages.

2. **Create AI Agent Node:**  
   - Add a LangChain agent node named `AI Agent`.  
   - Configure system prompt to:  
     ```
     You are a helpful assistant

     The current date is {{ $now }}.

     You are connected to a Google Calendar tool that lets you search for calendar events between two dates using the variables `afterDate` and `beforeDate` in ISO format (e.g., 2025-06-24T00:00:00Z). When the user asks for events, you must extract those fields and return them to the tool to make the query.
     ```  
   - Connect `When chat message received` node output to `AI Agent` input.

3. **Add Google Gemini Chat Model Node:**  
   - Add a LangChain language model node named `Google Gemini Chat Model`.  
   - Set modelName to `models/gemini-1.5-flash`.  
   - Connect this node as the `ai_languageModel` input for the `AI Agent`.

4. **Add Simple Memory Node:**  
   - Add a LangChain memoryBufferWindow node named `Simple Memory`.  
   - Connect `AI Agent` node’s `ai_memory` output to `Simple Memory` input, and connect `Simple Memory` output back to `AI Agent` node’s `ai_memory` input to maintain context.

5. **Create Google Calendar Add Event Node:**  
   - Add a Google Calendar Tool node named `Google Calendar`.  
   - Set calendar mode to `list` and select the target calendar ID (e.g., `7b358b08789959448cf50f4eba906fdbc4e260825dbaaffb5b4aac4ae80e2650@group.calendar.google.com`).  
   - Set operation to `create` (or equivalent for adding events).  
   - Set event fields with expressions:  
     - `start` = `{{$fromAI("startTime")}}`  
     - `end` = `{{$fromAI("endTime")}}`  
     - `summary` = `{{$fromAI("summary")}}`  
   - Connect `AI Agent` node’s `ai_tool` output to this node.

6. **Create Google Calendar Get Events Node:**  
   - Add another Google Calendar Tool node named `Google Calendar1`.  
   - Use the same calendar ID as above.  
   - Set operation to `getAll`.  
   - Set date filters with expressions:  
     - `timeMin` = `{{$fromAI("afterDate")}}`  
     - `timeMax` = `{{$fromAI("beforeDate")}}`  
   - Connect `AI Agent` node’s `ai_tool` output to this node as well.

7. **Set Credentials:**  
   - Configure Google Calendar credentials with OAuth2 for the target Google account/calendar.  
   - Ensure the LangChain nodes have proper API credentials for Google Gemini AI if required.

8. **Add Sticky Notes (Optional):**  
   - Create Sticky Note nodes around key areas to document purpose as per the original workflow.

9. **Connect All Nodes:**  
   - `When chat message received` → `AI Agent`  
   - `Google Gemini Chat Model` → `AI Agent` (language model input)  
   - `Simple Memory` → `AI Agent` (memory input/output)  
   - `AI Agent` → `Google Calendar` (add event)  
   - `AI Agent` → `Google Calendar1` (get events)

10. **Test:**  
    - Deploy the workflow and test with chat messages asking to add events or retrieve events using natural language.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                                                    |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates integration of Google Gemini 1.5 Flash model with Google Calendar API | Core concept: natural language to calendar event management                                                                      |
| Google Calendar API requires OAuth2 credentials with calendar scope                             | Ensure calendar permissions are granted and tokens refreshed as needed                                                           |
| For best accuracy, maintain conversation context using memoryBufferWindow                       | Allows multi-turn dialogues with AI to understand follow-up queries                                                              |
| Sticky notes provide good inline documentation for maintainers                                  | Use them to understand node responsibilities and to assist future modifications                                                  |
| Google Gemini 1.5 Flash is a powerful LLM suited for chat and extraction tasks                  | Model name: `models/gemini-1.5-flash`                                                                                            |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly respects current content policies and contains no illegal, offensive, or protected content. All manipulated data is legal and public.