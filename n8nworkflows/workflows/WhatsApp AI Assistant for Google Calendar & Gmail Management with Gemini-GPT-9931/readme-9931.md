WhatsApp AI Assistant for Google Calendar & Gmail Management with Gemini/GPT

https://n8nworkflows.xyz/workflows/whatsapp-ai-assistant-for-google-calendar---gmail-management-with-gemini-gpt-9931


# WhatsApp AI Assistant for Google Calendar & Gmail Management with Gemini/GPT

---

### 1. Workflow Overview

This workflow implements a **WhatsApp AI Assistant** designed to help users manage their **Google Calendar** and **Gmail** seamlessly through WhatsApp messages. It leverages the Google Gemini/GPT AI model to interpret natural language requests, interact with Google Calendar and Gmail APIs, and respond to users in real time.

**Target Use Cases:**  
- Scheduling, updating, and querying calendar events  
- Reading, summarizing, and sending emails  
- Accessing current date/time information  
- Clarifying ambiguous requests interactively  
- Personalized interactions based on user phone number  

**Logical Blocks:**

- **1.1 Input Reception & User Context Extraction:** Receives WhatsApp messages and extracts essential user context (message text, user phone number).
- **1.2 AI Processing & Memory:** Uses Google Gemini AI model as the core cognitive engine with a simple memory buffer to maintain conversational context per user.
- **1.3 Tool Integration & Execution:** Connects to Google Calendar and Gmail tools for fetching events/emails, creating/updating calendar events, and sending emails.
- **1.4 Output Delivery:** Sends AI-generated responses back to the user via WhatsApp.
- **1.5 Support & Setup Documentation:** Contains sticky notes with setup guidance, usage instructions, and help resources.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & User Context Extraction

**Overview:**  
This block receives incoming WhatsApp messages, extracts the message content and the user's phone number, and prepares data for processing by the AI agent.

**Nodes Involved:**  
- WhatsApp Trigger  
- Extract User Context  

**Node Details:**

- **WhatsApp Trigger**  
  - Type: Trigger node for WhatsApp Business Cloud API  
  - Configured to listen for incoming "messages" update events  
  - Outputs raw webhook JSON payload containing message and sender info  
  - Possible failures: webhook misconfiguration, authentication errors with Facebook API  

- **Extract User Context**  
  - Type: Set node  
  - Extracts key fields from webhook JSON:  
    - `chatInput`: message text body  
    - `sessionId`: user's phone number (used as session ID for AI memory)  
    - `userPhone`: user's phone number for reply routing  
  - Outputs enriched JSON for downstream nodes  
  - Failure modes: malformed webhook data, missing fields  

---

#### 1.2 AI Processing & Memory

**Overview:**  
This block processes the user's input using the Google Gemini AI language model wrapped by an AI Agent node, maintaining conversational context with a limited memory buffer.

**Nodes Involved:**  
- Simple Memory  
- Google Gemini Chat Model  
- AI Agent  

**Node Details:**

- **Simple Memory**  
  - Type: Memory Buffer node  
  - Maintains last ~10 interactions per user session (`sessionId` = phone number)  
  - Enables context-aware AI responses (pronoun resolution, follow-ups)  
  - Edge cases: memory overflow, session ID mismatches  

- **Google Gemini Chat Model**  
  - Type: AI language model node (Google Gemini)  
  - Uses Google PaLM API credentials  
  - Receives chat history and system prompt for contextual AI generation  
  - Replaceable with other models (OpenAI, Claude, etc.) if needed  
  - Potential failures: API quota limits, connectivity issues  

- **AI Agent**  
  - Type: LangChain Agent node (core AI engine)  
  - Configured with detailed system prompt defining operating principles and tool manifest  
  - Manages tool dispatch, clarification, confirmation, and execution logic  
  - Interfaces with all tool nodes (Calendar, Gmail, Date/Time)  
  - Handles user personalization via sessionId  
  - Retries on failure enabled  
  - Failure modes: expression errors, missing parameters, tool invocation errors  

---

#### 1.3 Tool Integration & Execution

**Overview:**  
This block comprises nodes representing various tools that the AI Agent can invoke to fulfill user requests, such as managing calendar events, sending emails, or fetching data.

**Nodes Involved:**  
- Date & Time  
- Get availability in a calendar in Google Calendar  
- Create an event in Google Calendar  
- Update an event in Google Calendar  
- Get many events in Google Calendar  
- Send a message in Gmail  
- Get many messages in Gmail  

**Node Details:**

- **Date & Time**  
  - Type: Date/Time Tool node  
  - Converts natural language or relative time expressions into ISO 8601 date/time strings  
  - Uses timezone info extracted from user input or asked via AI Agent  
  - Input: timezone string; Output: parsed date/time field  
  - Failure modes: ambiguous timezones, invalid date strings  

- **Get availability in a calendar in Google Calendar**  
  - Type: Google Calendar Tool node (resource: calendar)  
  - Checks user calendar availability between startTime and endTime  
  - Used to detect scheduling conflicts before event creation  
  - Requires Google Calendar OAuth2 credentials  
  - Edge cases: API rate limits, calendar access denied  

- **Create an event in Google Calendar**  
  - Type: Google Calendar Tool node (operation: create event)  
  - Creates new calendar events with details including summary, start/end, attendees, description, Google Meet conferencing  
  - Attendees added automatically if specified  
  - Uses Google Calendar OAuth2 credentials  
  - Failure modes: missing required fields, invalid date formats, permission denied  

- **Update an event in Google Calendar**  
  - Type: Google Calendar Tool node (operation: update event)  
  - Updates existing calendar events identified by eventId  
  - Supports partial updates to attendees, time, description, etc.  
  - Requires Google Calendar OAuth2 credentials  
  - Failure modes: invalid eventId, concurrency conflicts  

- **Get many events in Google Calendar**  
  - Type: Google Calendar Tool node (operation: get all events)  
  - Retrieves events within a date range specified by AI Agent  
  - Supports filtering by start and end times  
  - Requires Google Calendar OAuth2 credentials  
  - Edge cases: large data sets, API quota limits  

- **Send a message in Gmail**  
  - Type: Gmail Tool node (send message)  
  - Sends email based on recipient, subject, and body determined by AI Agent  
  - Uses Gmail OAuth2 credentials  
  - Failure modes: invalid recipient, OAuth token expiration  

- **Get many messages in Gmail**  
  - Type: Gmail Tool node (get all messages)  
  - Retrieves emails filtered by date range or sender, with limit and summary preferences set by AI Agent  
  - Uses Gmail OAuth2 credentials  
  - Failure modes: API rate limits, mailbox access issues  

---

#### 1.4 Output Delivery

**Overview:**  
Sends the AI Agent's generated response back to the user on WhatsApp, ensuring the message reaches the correct phone number.

**Nodes Involved:**  
- Send WhatsApp Response  

**Node Details:**

- **Send WhatsApp Response**  
  - Type: WhatsApp node (send operation)  
  - Sends text messages to the phone number extracted earlier (`userPhone`)  
  - Message content is the AI Agent output  
  - Uses WhatsApp Business Cloud API credentials  
  - Failure modes: message delivery errors, invalid recipient number  

---

#### 1.5 Support & Setup Documentation

**Overview:**  
This block contains sticky notes to assist developers and operators with setup instructions, node explanations, and support contacts.

**Nodes Involved:**  
- Setup Instructions (sticky note)  
- Need Help? (sticky note)  
- Multiple sticky notes explaining WhatsApp Trigger, AI Agent, Google Gemini Chat Model, Simple Memory, AI Tools  

**Details:**  
- Provide comprehensive stepwise guidance for configuring WhatsApp API, Google credentials, Google Gemini API key, and testing the workflow.  
- Contains links to external resources for API keys, help, and professional consulting.  
- Visual aids clarify node roles and expected behavior.  

---

### 3. Summary Table

| Node Name                            | Node Type                                   | Functional Role                                    | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                                                             |
|------------------------------------|---------------------------------------------|---------------------------------------------------|-----------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger                   | n8n-nodes-base.whatsAppTrigger               | Receives incoming WhatsApp messages                |                             | Extract User Context     | 📱 WhatsApp Trigger: Receives incoming WhatsApp messages from users. Configure Facebook App webhook accordingly.                        |
| Extract User Context               | n8n-nodes-base.set                           | Extracts message text and user phone number        | WhatsApp Trigger            | AI Agent                | 🔍 Extract User Context: Extracts message text, sessionId (phone), userPhone for responses.                                               |
| AI Agent                         | @n8n/n8n-nodes-langchain.agent               | Core AI engine: interprets input, manages tools    | Extract User Context         | Send WhatsApp Response   | 🤖 AI Agent: Brain of workflow, interprets messages, chooses tools, asks clarifications, confirms actions.                              |
| Send WhatsApp Response            | n8n-nodes-base.whatsApp                       | Sends response messages back to WhatsApp user      | AI Agent                    |                         | 📤 Send WhatsApp Response: Sends AI replies to correct user based on phone number.                                                      |
| Google Gemini Chat Model          | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI language model powering the assistant            | AI Agent (as languageModel) | AI Agent                | 🧠 Google Gemini Chat Model: AI model used (Google PaLM). Free and easy setup. API key at https://aistudio.google.com/apikey             |
| Simple Memory                    | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversational context per user          | AI Agent (as memory)        | AI Agent                | 📌 Simple Memory: Maintains last ~10 interactions per user sessionId for contextual understanding.                                       |
| Date & Time                     | n8n-nodes-base.dateTimeTool                   | Parses and converts natural language time input    | AI Agent (as ai_tool)       | AI Agent                |                                                                                                                                         |
| Get availability in a calendar in Google Calendar | n8n-nodes-base.googleCalendarTool        | Checks calendar availability for given time range | AI Agent (as ai_tool)       | AI Agent                |                                                                                                                                         |
| Create an event in Google Calendar | n8n-nodes-base.googleCalendarTool            | Creates new calendar events                         | AI Agent (as ai_tool)       | AI Agent                | 🛠️ AI Tools: Includes event creation on Google Calendar with conferencing options.                                                     |
| Update an event in Google Calendar | n8n-nodes-base.googleCalendarTool            | Updates existing calendar event                     | AI Agent (as ai_tool)       | AI Agent                | 🛠️ AI Tools: Allows event updates without creating new events.                                                                           |
| Get many events in Google Calendar | n8n-nodes-base.googleCalendarTool            | Retrieves calendar events in date range             | AI Agent (as ai_tool)       | AI Agent                | 🛠️ AI Tools: Lists calendar events for user inquiries.                                                                                   |
| Send a message in Gmail            | n8n-nodes-base.gmailTool                       | Sends emails from user Gmail                         | AI Agent (as ai_tool)       | AI Agent                | 🛠️ AI Tools: Sends Gmail messages crafted by AI based on user intent.                                                                   |
| Get many messages in Gmail         | n8n-nodes-base.gmailTool                       | Retrieves emails with optional filtering            | AI Agent (as ai_tool)       | AI Agent                | 🛠️ AI Tools: Checks inbox, filters emails by time or sender, provides detailed summaries.                                                |
| Setup Instructions                | n8n-nodes-base.stickyNote                      | Provides detailed setup guide for the workflow      |                             |                         | Setup Instructions: Complete guide for WhatsApp API, Google API credentials, Gemini API key, and testing instructions.                    |
| Need Help?                       | n8n-nodes-base.stickyNote                      | Provides help resources and contact information     |                             |                         | Need Help?: Links to tutorials, consulting, and troubleshooting resources at https://www.praneel.tech                                   |
| Sticky Note (WhatsApp Trigger)    | n8n-nodes-base.stickyNote                      | Explains WhatsApp Trigger node role                  |                             |                         | 📱 WhatsApp Trigger description                                                                                                         |
| Sticky Note (AI Agent)             | n8n-nodes-base.stickyNote                      | Explains AI Agent node role                           |                             |                         | 🤖 AI Agent description                                                                                                                 |
| Sticky Note (Google Gemini Chat)   | n8n-nodes-base.stickyNote                      | Explains Gemini Chat Model node                       |                             |                         | 🧠 Google Gemini Chat Model description                                                                                                |
| Sticky Note (Simple Memory)        | n8n-nodes-base.stickyNote                      | Explains Simple Memory role                           |                             |                         | 📌 Simple Memory description                                                                                                             |
| Sticky Note (AI Tools)             | n8n-nodes-base.stickyNote                      | Summarizes AI tools available                         |                             |                         | 🛠️ AI Tools overview                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Node Type: `WhatsApp Trigger`  
   - Configure with Facebook App credentials (App ID, App Secret, Access Token)  
   - Subscribe to "messages" update type  
   - Copy webhook URL for Meta Developers portal webhook configuration  

2. **Create Extract User Context Node**  
   - Node Type: `Set`  
   - Map fields from WhatsApp webhook JSON:  
     - `chatInput` = `{{$json.entry[0].changes[0].value.messages[0].text.body}}`  
     - `sessionId` = `{{$json.entry[0].changes[0].value.messages[0].from}}`  
     - `userPhone` = `{{$json.entry[0].changes[0].value.messages[0].from}}`  
   - Connect output of WhatsApp Trigger to this node  

3. **Create AI Agent Node**  
   - Node Type: `LangChain Agent`  
   - Configure with system prompt defining operational principles and tool manifest  
   - Enable retry on failure  
   - Connect output of Extract User Context here  

4. **Create Google Gemini Chat Model Node**  
   - Node Type: `LangChain LM Chat Google Gemini`  
   - Set up Google PaLM API credentials (create at https://aistudio.google.com/apikey)  
   - Connect as `ai_languageModel` input of AI Agent  

5. **Create Simple Memory Node**  
   - Node Type: `LangChain Memory Buffer Window`  
   - Set context window length to 10  
   - Connect as `ai_memory` input to AI Agent  

6. **Create Date & Time Node**  
   - Node Type: `Date & Time Tool`  
   - Receive `timezone` from AI Agent input for parsing natural language times  
   - Connect as `ai_tool` input to AI Agent  

7. **Setup Google Calendar Nodes:**  
   - **Get availability in a calendar**  
     - Node Type: Google Calendar Tool, resource `calendar`  
     - Parameters accept `Start_Time` and `End_Time` from AI Agent  
     - Connect as `ai_tool` input to AI Agent  
   - **Create an event**  
     - Node Type: Google Calendar Tool, operation `create`  
     - Configure calendar selection and OAuth2 credentials  
     - Accept parameters: start, end, summary, attendees, description, conferencing  
     - Connect as `ai_tool` input to AI Agent  
   - **Update an event**  
     - Node Type: Google Calendar Tool, operation `update`  
     - Accept parameters: eventId, update fields  
     - Connect as `ai_tool` input to AI Agent  
   - **Get many events**  
     - Node Type: Google Calendar Tool, operation `getAll`  
     - Accept timeMin and timeMax parameters  
     - Connect as `ai_tool` input to AI Agent  

8. **Setup Gmail Nodes:**  
   - **Send a message**  
     - Node Type: Gmail Tool, operation `send`  
     - Accept recipient, subject, message body from AI Agent  
     - Configure OAuth2 credentials for Gmail  
     - Connect as `ai_tool` input to AI Agent  
   - **Get many messages**  
     - Node Type: Gmail Tool, operation `getAll`  
     - Accept limit, filters (date range, sender) from AI Agent  
     - Connect as `ai_tool` input to AI Agent  

9. **Create Send WhatsApp Response Node**  
   - Node Type: WhatsApp node (send operation)  
   - Use same WhatsApp credentials as WhatsApp Trigger  
   - Set recipient phone number as expression from `userPhone` extracted earlier  
   - Set text message as AI Agent output field `output`  
   - Connect AI Agent output to this node  

10. **Add Sticky Notes**  
    - Create sticky notes with setup instructions, node explanations, and help URLs for maintainability and reference  

11. **Finalize**  
    - Verify all credentials are authorized and active  
    - Test each tool node independently with sample data  
    - Activate workflow and test end-to-end via WhatsApp messages  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------|
| Full WhatsApp Business Cloud API setup requires creating a Facebook App, adding WhatsApp product, generating permanent tokens, and configuring webhooks correctly. | Setup Instructions sticky note          |
| Google Gemini (PaLM) API key is free and easy to obtain at https://aistudio.google.com/apikey, and is used as the AI language model in this workflow.          | Google Gemini Chat Model sticky note    |
| Conversation context is stored per phone number sessionId using a simple memory buffer to enable natural dialogues referencing prior messages.                 | Simple Memory sticky note                |
| AI Agent strictly follows tool manifest and operating principles, including asking clarifying questions and confirming actions before execution.               | AI Agent sticky note                     |
| Visit https://www.praneel.tech for tutorials, consulting, and troubleshooting assistance related to WhatsApp API and Google API integrations.                   | Need Help? sticky note                   |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a powerful integration and automation tool. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.

---