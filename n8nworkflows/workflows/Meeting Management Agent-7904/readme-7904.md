Meeting Management Agent

https://n8nworkflows.xyz/workflows/meeting-management-agent-7904


# Meeting Management Agent

### 1. Workflow Overview

The **Meeting Booking Agent** is an automated scheduling assistant designed to interact with users via Telegram, understand natural language meeting requests, check Google Calendar for availability, create or update meetings, and send email notifications via Gmail. It leverages AI capabilities to interpret user intents and manages meeting logistics end-to-end with conflict prevention and natural language understanding.

**Target Use Cases:**  
- Scheduling meetings by natural language commands sent through Telegram  
- Querying calendar events and availability  
- Avoiding double-bookings and offering alternatives  
- Sending meeting invitations or confirmations via Gmail  

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving user messages from Telegram  
- **1.2 AI Processing:** Natural language understanding and decision-making using AI Agent and OpenAI models  
- **1.3 Calendar Management:** Reading, creating, updating, and deleting Google Calendar events  
- **1.4 Email Notification:** Sending meeting details or invitations via Gmail  
- **1.5 Conversation Memory:** Maintaining session context for ongoing chats  
- **1.6 Date & Time Handling:** Calculating exact dates and times from natural language expressions  
- **1.7 User Response:** Sending responses back to Telegram users  
- **1.8 Setup and Documentation:** Sticky notes containing setup guides and video tutorial links  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures incoming messages from Telegram users to trigger the workflow.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  

  **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Receives Telegram updates, specifically new messages  
  - Configuration: Watches for "message" update type only  
  - Input: Incoming Telegram messages via webhook  
  - Output: Passes message text and metadata to the AI Agent node  
  - Version: 1.2  
  - Edge Cases: Telegram webhook misconfiguration, bot token errors, message format issues  
  - Notes: Requires Telegram bot token setup in credentials  

#### 1.2 AI Processing

- **Overview:**  
  Processes the input message using an AI agent powered by LangChain and OpenAI, applying business logic for scheduling meetings.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI  
  - Memory  
  - Date & Time

- **Node Details:**  

  **AI Agent**  
  - Type: LangChain Agent node  
  - Role: Central AI decision-maker interpreting text, orchestrating tool usage (calendar checks, date calculations, email sending)  
  - Configuration:  
    - Input text: Extracted from Telegram message text  
    - System message: Detailed instructions for scheduling logic, conflict checking, tool usage priorities, and user-friendly dialogue style  
    - Tools referenced: Date & Time, Google Calendar (Get/Update/Create/Delete), Gmail  
  - Input: Telegram Trigger output (message text), Memory, OpenAI, Date & Time, Google Calendar, Gmail nodes (via ai_tool and ai_memory connections)  
  - Output: Response text to send back to user  
  - Version: 2  
  - Edge Cases: Misinterpretation, AI model rate limits, API key issues, downstream tool failures  
  - Sub-workflow: Acts as a coordination layer invoking other nodes as tools  

  **OpenAI**  
  - Type: LangChain OpenAI Chat Model node  
  - Role: Provides language model responses to the AI Agent  
  - Configuration: Model set to "gpt-4.1-mini" optimized for chat interaction  
  - Input: Prompt text from AI Agent  
  - Output: Language model response used by AI Agent  
  - Version: 1.2  
  - Edge Cases: API key invalid, rate limits, model unavailability  

  **Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversation context per Telegram chat ID to enable coherent multi-turn dialogue  
  - Configuration: Uses Telegram chat ID (`message.chat.id`) as session key  
  - Input: Chat history from AI Agent  
  - Output: Updated memory state passed back to AI Agent  
  - Version: 1.3  
  - Edge Cases: Memory overflow, session key mismatch  

  **Date & Time**  
  - Type: DateTime Tool  
  - Role: Computes exact dates and times from natural language expressions (e.g., "tomorrow", "next Monday")  
  - Configuration: Timezone set to "Asia/Dhaka" for consistent scheduling  
  - Input: Date-related queries from AI Agent  
  - Output: Precise date/time values returned to AI Agent  
  - Version: 2  
  - Edge Cases: Timezone misconfiguration, ambiguous date expressions  

#### 1.3 Calendar Management

- **Overview:**  
  Manages Google Calendar events by reading availability, creating, updating, or deleting events as instructed by the AI Agent.

- **Nodes Involved:**  
  - Get (Google Calendar GetAll)  
  - Create (Google Calendar Create Event)  
  - Update (Google Calendar Update Event)  
  - Delete (Google Calendar Delete Event)

- **Node Details:**  

  **Get**  
  - Type: Google Calendar Tool (getAll)  
  - Role: Queries calendar events within specified time ranges to check availability or list meetings  
  - Configuration:  
    - Calendar ID preset to a specific calendar (likely user’s primary or a shared calendar)  
    - Time range and limit parameters dynamically set by AI Agent overrides  
  - Input: Parameters from AI Agent (timeMin, timeMax, limit)  
  - Output: List of events matching criteria  
  - Version: 1.3  
  - Edge Cases: Invalid calendar ID, API quota exceeded, authentication failure  

  **Create**  
  - Type: Google Calendar Tool (create)  
  - Role: Creates new events if no scheduling conflict exists  
  - Configuration:  
    - Start and end times, summary, attendees, and description set via AI Agent overrides  
    - Calendar ID same as Get node  
  - Input: Meeting details from AI Agent  
  - Output: Confirmation of event creation  
  - Version: 1.3  
  - Edge Cases: Overlapping events, invalid time ranges, attendee email format errors  

  **Update**  
  - Type: Google Calendar Tool (update)  
  - Role: Modifies existing events based on AI Agent instructions  
  - Configuration: Event ID and update fields provided by AI Agent  
  - Input: Event ID and update data  
  - Output: Updated event confirmation  
  - Version: 1.3  
  - Edge Cases: Event not found, update conflicts  

  **Delete**  
  - Type: Google Calendar Tool (delete)  
  - Role: Removes events when cancellation is requested or conflicts arise  
  - Configuration: Event ID supplied by AI Agent  
  - Input: Event ID  
  - Output: Deletion confirmation  
  - Version: 1.3  
  - Edge Cases: Event already deleted, permissions issues  

#### 1.4 Email Notification

- **Overview:**  
  Sends meeting invitations or confirmation emails to attendees via Gmail.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**  

  **Gmail**  
  - Type: Gmail Tool node  
  - Role: Sends emails with meeting details composed by AI Agent  
  - Configuration:  
    - Recipient email(s), subject, and message body dynamically set by AI Agent overrides  
    - Email type set to plain text  
  - Input: Email parameters from AI Agent  
  - Output: Email sent status  
  - Version: 2.1  
  - Edge Cases: Authentication issues, invalid recipient addresses, email delivery failures  

#### 1.5 Conversation Memory

- **Overview:**  
  Maintains session context for users to enable natural, coherent multi-turn conversations.

- **Nodes Involved:**  
  - Memory

- **Node Details:** See section 1.2 AI Processing above.

#### 1.6 Date & Time Handling

- **Overview:**  
  Converts relative or ambiguous date/time expressions into absolute timestamps to ensure accurate scheduling.

- **Nodes Involved:**  
  - Date & Time

- **Node Details:** See section 1.2 AI Processing above.

#### 1.7 User Response

- **Overview:**  
  Sends AI-generated responses back to the user on Telegram.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**  

  **Telegram**  
  - Type: Telegram node (send message)  
  - Role: Delivers text responses from AI Agent back to the user’s chat  
  - Configuration:  
    - Text content dynamically set from AI Agent's output  
    - Chat ID taken from the original Telegram Trigger message  
    - Attribution disabled for clean messages  
  - Input: AI Agent output text  
  - Output: Message sent confirmation  
  - Version: 1.2  
  - Edge Cases: Invalid chat ID, Telegram API downtime, message size limits  

#### 1.8 Setup and Documentation

- **Overview:**  
  Contains detailed instructions and a video tutorial link to facilitate setup and usage.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note3

- **Node Details:**  

  **Sticky Note**  
  - Type: Sticky Note node  
  - Role: Provides a comprehensive setup guide with stepwise instructions and links to external resources (Telegram Bot creation, OpenAI keys, Google Calendar, Gmail, timezone settings)  
  - Content: Detailed textual guide covering all main integrations and configuration steps  
  - Position: Prominently placed for user reference  

  **Sticky Note3**  
  - Type: Sticky Note node  
  - Role: Shares a YouTube tutorial link with a thumbnail preview to help users start quickly  
  - Content: Link to a step-by-step video on the Meeting Booking n8n Agent  
  - Position: Near workflow start for easy access  

---

### 3. Summary Table

| Node Name       | Node Type                              | Functional Role                        | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                                            |
|-----------------|--------------------------------------|-------------------------------------|----------------------|----------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger| Telegram Trigger                     | Receive user messages from Telegram | (Webhook input)       | AI Agent             | See setup guide for Telegram Bot connection                                                                                           |
| AI Agent        | LangChain Agent                     | Interpret message, orchestrate tools| Telegram Trigger, Memory, OpenAI, Date & Time, Google Calendar, Gmail | Telegram           | Core AI logic with system message enforcing scheduling rules                                                                          |
| OpenAI          | LangChain OpenAI Chat Model          | Generate AI text responses           | AI Agent             | AI Agent             | Model: gpt-4.1-mini                                                                                                                    |
| Memory          | LangChain Memory Buffer              | Maintain conversation context        | AI Agent             | AI Agent             | Session key: Telegram chat ID                                                                                                         |
| Date & Time     | DateTime Tool                       | Calculate exact date/time values     | AI Agent             | AI Agent             | Timezone: Asia/Dhaka                                                                                                                   |
| Get             | Google Calendar Tool (getAll)         | Query calendar events for conflicts | AI Agent             | AI Agent             | Calendar ID preset; dynamic time range                                                                                                |
| Create          | Google Calendar Tool (create)         | Create new meetings                  | AI Agent             | AI Agent             | Calendar ID preset; uses AI-provided meeting details                                                                                   |
| Update          | Google Calendar Tool (update)         | Update existing meetings             | AI Agent             | AI Agent             | Calendar ID preset; event ID and fields from AI                                                                                        |
| Delete          | Google Calendar Tool (delete)         | Delete meetings                     | AI Agent             | AI Agent             | Calendar ID preset; event ID from AI                                                                                                  |
| Gmail           | Gmail Tool                          | Send meeting emails                  | AI Agent             |                      | Requires Gmail OAuth2 credentials                                                                                                     |
| Telegram        | Telegram Node                      | Send messages back to users          | AI Agent             |                      | Uses chat ID from Telegram Trigger                                                                                                    |
| Sticky Note     | Sticky Note                        | Setup instructions                   | None                 | None                 | Detailed setup guide with links for Telegram, OpenAI, Google Calendar, Gmail, timezone, and AI Agent rules                            |
| Sticky Note3    | Sticky Note                        | Video tutorial link                  | None                 | None                 | YouTube tutorial: https://youtu.be/IWDGfZyN5to                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Monitor "message" update type  
   - Credential: Provide Telegram Bot token from @BotFather  
   - Connect output to AI Agent node  

2. **Create AI Agent Node**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text input: `={{ $json.message.text }}` from Telegram Trigger  
     - System message: Detailed instructions for scheduling, conflict checking, tool usage, and friendly responses (copy from original)  
   - Connect input from Telegram Trigger and Memory, OpenAI, Date & Time, Google Calendar, Gmail nodes as AI tools/memory  
   - Connect output to Telegram node  

3. **Create OpenAI Node**  
   - Type: LangChain OpenAI Chat  
   - Parameters: Model set to "gpt-4.1-mini"  
   - Credential: OpenAI API key  
   - Connect input/output to AI Agent as language model  

4. **Create Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Parameters: Session Key: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Connect input/output to AI Agent as AI memory  

5. **Create Date & Time Node**  
   - Type: DateTime Tool  
   - Parameters: Timezone set to your local zone (e.g., "Asia/Dhaka")  
   - Connect input/output to AI Agent as AI tool  

6. **Create Google Calendar Nodes**  
   - Create **Get** node:  
     - Type: Google Calendar Tool (getAll)  
     - Parameters: Calendar ID set to your calendar, timeMin/timeMax and limit dynamically received from AI Agent overrides  
     - Credential: Google Calendar OAuth2  
     - Connect as AI tool to AI Agent  
   - Create **Create** node:  
     - Type: Google Calendar Tool (create)  
     - Parameters: Start, End, Summary, Attendees, Description from AI Agent overrides  
     - Credential: Google Calendar OAuth2  
     - Connect as AI tool to AI Agent  
   - Create **Update** node:  
     - Type: Google Calendar Tool (update)  
     - Parameters: Event ID and update fields from AI Agent overrides  
     - Credential: Google Calendar OAuth2  
     - Connect as AI tool to AI Agent  
   - Create **Delete** node:  
     - Type: Google Calendar Tool (delete)  
     - Parameters: Event ID from AI Agent overrides  
     - Credential: Google Calendar OAuth2  
     - Connect as AI tool to AI Agent  

7. **Create Gmail Node**  
   - Type: Gmail Tool  
   - Parameters: SendTo, Subject, Message dynamically from AI Agent overrides  
   - Credential: Gmail OAuth2  
   - Connect as AI tool to AI Agent  

8. **Create Telegram Node (Send Message)**  
   - Type: Telegram  
   - Parameters:  
     - Text: `={{ $json.output }}` from AI Agent output  
     - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
     - Disable attribution  
   - Connect input from AI Agent  

9. **Add Sticky Note Nodes for Documentation**  
   - Add one large sticky note with detailed setup instructions (copy content from original)  
   - Add another sticky note with YouTube tutorial link and thumbnail  

10. **Ensure all credentials are properly configured:**  
    - Telegram Bot token in Telegram Trigger and Telegram nodes  
    - OpenAI API key in OpenAI node  
    - Google Calendar OAuth2 in all calendar nodes  
    - Gmail OAuth2 in Gmail node  

11. **Set workflow active and test by sending messages to Telegram bot**  
    - Verify AI responds naturally, calendar is checked, events created, and emails sent  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Setup guide covers Telegram Bot creation, OpenAI key setup, Google Calendar linking, Gmail connection, timezone configuration, and AI Agent rules | Provided in the large sticky note node in the workflow                                                                    |
| Step-by-step video tutorial on meeting booking agent with n8n on YouTube                                                                             | https://youtu.be/IWDGfZyN5to                                                                                              |
| AI Agent system message enforces strict scheduling logic including date accuracy, conflict checks, and tool usage for professional responses      | Embedded in AI Agent node parameters                                                                                      |
| Timezone set to Asia/Dhaka but should be adjusted to user’s local timezone for accurate date handling                                              | Date & Time node configuration                                                                                            |
| Workflow integrates multiple services seamlessly, requiring valid credentials and OAuth2 permissions for Google and Gmail APIs                     | Credentials must be configured before workflow activation                                                                  |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.