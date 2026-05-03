Manage Google Calendar & Gmail with a Gemini/GPT Powered AI Assistant

https://n8nworkflows.xyz/workflows/manage-google-calendar---gmail-with-a-gemini-gpt-powered-ai-assistant-8159


# Manage Google Calendar & Gmail with a Gemini/GPT Powered AI Assistant

### 1. Workflow Overview

This workflow, titled **"Google Calendar+GMail Manager"**, implements an AI-powered personal assistant designed to manage a user's Google Calendar and Gmail interactions through natural language commands. It leverages a large language model (Google Gemini) integrated with a LangChain AI Agent that interprets and translates user requests into specific API calls to Google Calendar and Gmail services.

**Target use cases** include:  
- Scheduling, updating, and querying calendar events.  
- Sending and reading emails with contextual filters.  
- Managing meeting invites and attendee notifications.  
- Handling time-related queries with timezone awareness.

The workflow is organized into the following logical blocks:

- **1.1 User Interaction & AI Processing:** Receives user input via chat, processes context and memory, and uses an LLM-powered AI Agent to interpret intent and select tools.  
- **1.2 Time and Date Handling:** Converts natural language time expressions to ISO 8601 timestamps and retrieves current time in specified timezones.  
- **1.3 Calendar Management:** Fetches events, checks availability, creates new events, and updates existing events in Google Calendar.  
- **1.4 Email Management:** Reads emails with filtering options and sends emails as requested.  
- **1.5 Memory and Context Management:** Maintains short-term memory of recent interactions to provide conversational continuity.

---

### 2. Block-by-Block Analysis

#### 2.1 User Interaction & AI Processing

**Overview:**  
This block captures user input, processes it through an AI language model and an AI Agent node that interprets commands, manages dialog flow, and selects appropriate tools based on the user's requests.

**Nodes Involved:**  
- Chat Node  
- Google Gemini Chat Model  
- AI Agent  
- Simple Memory

##### Node Details:

- **Chat Node**  
  - *Type:* `@n8n/n8n-nodes-langchain.chatTrigger`  
  - *Role:* Entry point for user messages; triggers workflow on new chat input.  
  - *Configuration:* Default options; listens for chat messages.  
  - *Input:* External chat input.  
  - *Output:* Passes user message to AI Agent.  
  - *Edge Cases:* Missing input; malformed messages.  
  - *Notes:* Can be configured to accept input from other messenger apps (Discord, WhatsApp, Telegram) as per sticky note.

- **Google Gemini Chat Model**  
  - *Type:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  - *Role:* Provides natural language understanding and generation via Google Gemini LLM.  
  - *Configuration:* Default language model options; can be swapped with another LLM (e.g., OpenAI, Claude).  
  - *Input:* User message from Chat Node, context from memory.  
  - *Output:* LLM-generated interpretation or response.  
  - *Edge Cases:* API key or quota issues; network errors.  
  - *Notes:* Setup requires an AI API key from Google AI Studio ([AI API Key](https://aistudio.google.com/apikey)).

- **AI Agent**  
  - *Type:* `@n8n/n8n-nodes-langchain.agent`  
  - *Role:* Core cognitive engine that interprets user requests and orchestrates tool calls. Enforces strict protocols on tool usage, requests clarifications, confirms actions before execution.  
  - *Configuration:*  
    - System message defines assistant behavior and tool manifest usage.  
    - Implements multi-step processes for scheduling meetings and sending invites.  
  - *Input:* Outputs from Chat Node, Google Gemini Model, and Simple Memory.  
  - *Output:* Triggers relevant tool nodes with structured parameters.  
  - *Edge Cases:* Ambiguous user requests; missing parameters; authorization errors on tools; API call failures.  
  - *Notes:* Sticky note highlights this as the workflow “brain.”

- **Simple Memory**  
  - *Type:* `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - *Role:* Maintains a rolling context window of last 10 interactions to preserve conversational continuity and references.  
  - *Configuration:* Context window length set to 10 messages.  
  - *Input:* Conversation history.  
  - *Output:* Context for AI Agent and LLM to maintain state.  
  - *Edge Cases:* Context overflow; irrelevant memory entries.  
  - *Notes:* Sticky note explains its role for short-term memory.

---

#### 2.2 Time and Date Handling

**Overview:**  
Converts natural language time expressions into precise ISO 8601 timestamps and retrieves current date/time in specified timezones, essential for correctly scheduling and querying events.

**Nodes Involved:**  
- Date & Time

##### Node Details:

- **Date & Time**  
  - *Type:* `n8n-nodes-base.dateTimeTool`  
  - *Role:* Parses and generates formatted date-time strings based on user queries, including timezone conversion.  
  - *Configuration:*  
    - Accepts timezone input from AI Agent (dynamic).  
    - Outputs ISO 8601 formatted timestamp or other requested date/time fields.  
  - *Input:* Natural language time queries from AI Agent.  
  - *Output:* Structured time data for calendar and email tool parameters.  
  - *Edge Cases:* Invalid or missing timezone identifiers; ambiguous natural language expressions.  
  - *Notes:* Used to resolve relative dates like “today,” “tomorrow,” or “this week.”

---

#### 2.3 Calendar Management

**Overview:**  
Handles all Google Calendar related operations: retrieving events, verifying availability for scheduling, creating new events with optional attendees and Google Meet link, and updating existing events.

**Nodes Involved:**  
- Get many events in Google Calendar  
- Get availability in a calendar in Google Calendar  
- Create an event in Google Calendar  
- Update an event in Google Calendar

##### Node Details:

- **Get many events in Google Calendar**  
  - *Type:* `n8n-nodes-base.googleCalendarTool`  
  - *Role:* Retrieves a list of calendar events within a specified date range.  
  - *Configuration:*  
    - Uses calendar “p70157015@gmail.com” (pre-configured).  
    - Input parameters `timeMin` and `timeMax` dynamically set from AI Agent.  
  - *Input:* Date range from AI Agent.  
  - *Output:* List of events for availability checking or schedule summaries.  
  - *Edge Cases:* API authorization errors; invalid date ranges; empty calendar.  

- **Get availability in a calendar in Google Calendar**  
  - *Type:* `n8n-nodes-base.googleCalendarTool`  
  - *Role:* Checks user availability by querying calendar events for a given timeslot to detect conflicts.  
  - *Configuration:*  
    - Similar calendar and dynamic `timeMin`/`timeMax` parameters.  
  - *Input:* Proposed start and end times from AI Agent.  
  - *Output:* Boolean or event list indicating availability.  
  - *Edge Cases:* Overlapping events; timezone mismatches; API errors.  

- **Create an event in Google Calendar**  
  - *Type:* `n8n-nodes-base.googleCalendarTool`  
  - *Role:* Schedules a new event with title, start/end times, optional attendees, description, and Google Meet conferencing.  
  - *Configuration:*  
    - Calendar fixed as above.  
    - Attendees added dynamically from AI Agent parameters.  
    - Conference data set to Hangouts Meet.  
    - Option to use default reminders configurable by AI Agent.  
  - *Input:* Event details from AI Agent after availability check and user confirmation.  
  - *Output:* Confirmation of event creation, event ID for follow-up operations.  
  - *Edge Cases:* Missing required fields; invalid attendee emails; API quota limits.

- **Update an event in Google Calendar**  
  - *Type:* `n8n-nodes-base.googleCalendarTool`  
  - *Role:* Modifies existing events with updated details.  
  - *Configuration:*  
    - Event ID parameter required.  
    - Calendar fixed.  
    - Update fields passed dynamically (empty in provided config, but designed for extensibility).  
  - *Input:* Event ID and updated data from AI Agent.  
  - *Output:* Confirmation of update success.  
  - *Edge Cases:* Invalid event ID; concurrent modification conflicts; permission errors.

---

#### 2.4 Email Management

**Overview:**  
Manages sending emails and retrieving emails from Gmail with filtering capabilities. Supports composing messages, sending invitations, and summarizing inbox content based on user queries.

**Nodes Involved:**  
- Send a message in Gmail  
- Get many messages in Gmail

##### Node Details:

- **Send a message in Gmail**  
  - *Type:* `n8n-nodes-base.gmailTool`  
  - *Role:* Composes and sends emails with recipient, subject, and body specified by AI Agent.  
  - *Configuration:*  
    - Recipient, subject, and message body dynamically set via AI Agent expressions.  
    - Option to append attribution enabled.  
  - *Input:* Email fields from AI Agent post event creation or direct user request.  
  - *Output:* Email send confirmation or error.  
  - *Edge Cases:* Invalid email addresses; SMTP errors; message content errors.

- **Get many messages in Gmail**  
  - *Type:* `n8n-nodes-base.gmailTool`  
  - *Role:* Retrieves multiple emails with optional filters such as date range and sender.  
  - *Configuration:*  
    - Limit and filter query parameters set dynamically by AI Agent, supporting time frames and sender filters.  
    - Returns detailed summaries, not mere brief snippets.  
  - *Input:* Query parameters from AI Agent.  
  - *Output:* List of matching emails.  
  - *Edge Cases:* Large inboxes causing timeouts; invalid query syntax; authorization failures.

---

### 3. Summary Table

| Node Name                         | Node Type                                 | Functional Role                        | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                                                |
|----------------------------------|-------------------------------------------|-------------------------------------|----------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Chat Node                        | @n8n/n8n-nodes-langchain.chatTrigger     | Receives user input as chat trigger | External chat input         | AI Agent                  | ### 🗣️ Chat Node If you wish to, you can switch to a different Messenger app like discord, Whatsapp and Telegram                         |
| Google Gemini Chat Model         | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM processing of user input         | AI Agent                   | AI Agent                  | ### 🧠 Google Gemini Chat Model (or any LLM) The AI language model that powers the assistant. You can swap Gemini with OpenAI, Claude, or another model. (I have used Gemini since it's free and easy to setup here is the link to get your own [AI API Key](https://aistudio.google.com/apikey) |
| Simple Memory                   | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains short-term memory context | AI Agent                   | AI Agent                  | ### 📌 Simple Memory Keeps short-term context of the last ~10 interactions so the agent remembers what “it” refers to in your requests.     |
| AI Agent                        | @n8n/n8n-nodes-langchain.agent            | Core AI logic & orchestrator         | Chat Node, Gemini Model, Simple Memory | Google Calendar & Gmail nodes | ### 🤖 AI Agent The “brain” of the workflow. Interprets your requests and chooses the right tool. Asks for clarification if details are missing and confirms before important actions. |
| Send a message in Gmail          | n8n-nodes-base.gmailTool                   | Sends emails                         | AI Agent                   | -                         | ### Tools - Sends emails from your Gmail account. Requires recipient, subject, and body. Generates clear, professional text.               |
| Get many messages in Gmail       | n8n-nodes-base.gmailTool                   | Fetches emails with filters         | AI Agent                   | -                         | ### Tools - Checks your inbox. Can filter by timeframe or sender. Returns detailed summaries, not just one-liners.                         |
| Get many events in Google Calendar | n8n-nodes-base.googleCalendarTool          | Retrieves calendar events            | AI Agent                   | -                         | ### Tools - Lists your events for a chosen date range. Useful for “What’s on my schedule this week?”                                         |
| Date & Time                     | n8n-nodes-base.dateTimeTool                | Converts natural language time to ISO timestamps | AI Agent                   | AI Agent                  | ### Tools - Converts natural phrases like “tomorrow at 3 PM” into exact ISO date-time values.                                               |
| Get availability in a calendar in Google Calendar | n8n-nodes-base.googleCalendarTool          | Checks calendar availability        | AI Agent                   | -                         | ### Tools - Checks if you’re free during a specific time slot. Prevents double-booking before scheduling.                                   |
| Create an event in Google Calendar | n8n-nodes-base.googleCalendarTool          | Creates calendar event               | AI Agent                   | -                         | ### Tools - Schedules a new meeting. Adds title, start/end times, attendees, description, and Google Meet link.                             |
| Update an event in Google Calendar | n8n-nodes-base.googleCalendarTool          | Updates existing calendar event     | AI Agent                   | -                         | ### Tools - Edits an existing meeting. Change time, attendees, or details without creating a new event.                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Purpose: To receive user chat input and trigger the workflow.

2. **Add Google Gemini Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Purpose: To process natural language using Google Gemini LLM.  
   - Configure with your Google AI Studio API key for authentication.

3. **Add Simple Memory Node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Parameters: Set `contextWindowLength` to 10.  
   - Purpose: Maintain last 10 interactions for conversational context.

4. **Add AI Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - Set the `systemMessage` with detailed assistant behavior, tool manifest, and multi-step workflow logic as in the original system prompt (see section 1).  
   - Purpose: Interpret user requests, manage dialog, and orchestrate tool usage strictly.  
   - Connect inputs from Chat Node (main), Google Gemini Chat Model (ai_languageModel), and Simple Memory (ai_memory).  
   - Connect outputs to all tool nodes as `ai_tool` inputs.

5. **Add Date & Time Node**  
   - Type: `n8n-nodes-base.dateTimeTool`  
   - Parameters:  
     - Timezone dynamically passed from AI Agent.  
     - Output field name as needed by AI Agent.  
   - Purpose: Convert natural language time expressions to ISO 8601 strings.

6. **Add Google Calendar Nodes:**

   a. **Get many events in Google Calendar**  
      - Type: `n8n-nodes-base.googleCalendarTool`  
      - Operation: `getAll`  
      - Calendar: Select user's calendar (e.g., `p70157015@gmail.com`)  
      - Parameters: `timeMin` and `timeMax` set dynamically from AI Agent.

   b. **Get availability in a calendar in Google Calendar**  
      - Type: `n8n-nodes-base.googleCalendarTool`  
      - Operation: `getAll` or availability check as per API  
      - Calendar: Same as above  
      - Parameters: `timeMin` and `timeMax` dynamically set for proposed event time.

   c. **Create an event in Google Calendar**  
      - Type: `n8n-nodes-base.googleCalendarTool`  
      - Operation: `create`  
      - Calendar: Same as above  
      - Parameters:  
        - `start` and `end` times dynamically passed.  
        - `summary` (event title), `description`, and `attendees` array set dynamically.  
        - Enable Google Meet conferencing via `conferenceDataUi`.  
        - Optionally use default reminders.

   d. **Update an event in Google Calendar**  
      - Type: `n8n-nodes-base.googleCalendarTool`  
      - Operation: `update`  
      - Calendar: Same as above  
      - Parameters:  
        - `eventId` dynamically provided.  
        - Update fields as needed.

7. **Add Gmail Nodes:**

   a. **Send a message in Gmail**  
      - Type: `n8n-nodes-base.gmailTool`  
      - Operation: send message  
      - Parameters:  
        - `recipient` (email address) from AI Agent.  
        - `subject` and `body` from AI Agent, ensuring professional content.  
        - Enable attribution append option.

   b. **Get many messages in Gmail**  
      - Type: `n8n-nodes-base.gmailTool`  
      - Operation: get all or filtered  
      - Parameters:  
        - `limit` and `filters` (e.g., by date range, sender) dynamically set from AI Agent.

8. **Connect Nodes:**

   - Chat Node triggers AI Agent.  
   - AI Agent receives inputs from Chat Node, Google Gemini Chat Model, and Simple Memory.  
   - AI Agent outputs trigger the tool nodes as needed (Date & Time, Gmail, Google Calendar nodes).  
   - Responses from tools feed back to AI Agent for further processing or confirmation.

9. **Configure Credentials:**

   - Provide OAuth2 credentials for Google Calendar and Gmail with necessary scopes (read/write calendar events, send/read emails).  
   - Configure Google Gemini API access with valid API key.

10. **Optional: Add Sticky Notes**

    - Add explanatory sticky notes near key nodes for documentation and user guidance as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini as the default AI language model but is designed to be interchangeable with OpenAI, Claude, or others.                                  | [AI API Key](https://aistudio.google.com/apikey)                                                  |
| The AI Agent strictly follows a tool manifest and operational protocols to ensure safe, precise, and user-confirmed operations.                                         | System prompt within AI Agent node                                                                |
| Chat input can be sourced from alternative messenger apps like Discord, WhatsApp, and Telegram by reconfiguring the Chat Node accordingly.                             | Sticky note near Chat Node                                                                        |
| The workflow maintains short-term conversational memory (~10 interactions) to provide context-aware assistance.                                                        | Sticky note near Simple Memory node                                                               |
| The workflow includes multi-step scheduling logic to check calendar conflicts before creating events and sending invitations, preventing double bookings.              | Defined in AI Agent's system message and operational steps                                        |
| Google Calendar events created include Google Meet conferencing links by default for convenience.                                                                       | Create event node parameters                                                                       |
| Gmail email sending requires professional, clear content with relevant subjects matching the email body for user clarity and professionalism.                          | Email node parameters and AI Agent protocols                                                     |

---

**Disclaimer:** The provided content originates exclusively from an n8n automated workflow. The workflow respects all current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.