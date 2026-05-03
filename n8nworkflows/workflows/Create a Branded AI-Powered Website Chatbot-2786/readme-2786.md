Create a Branded AI-Powered Website Chatbot

https://n8nworkflows.xyz/workflows/create-a-branded-ai-powered-website-chatbot-2786


# Create a Branded AI-Powered Website Chatbot

### 1. Workflow Overview

This workflow implements a branded AI-powered chatbot for websites, designed to engage visitors intelligently and facilitate appointment scheduling and lead capture. It integrates OpenAI’s GPT-4 model for natural conversation, Microsoft Outlook calendar for availability checks and appointment booking, and includes fallback human handoff via email.

The workflow is logically divided into these blocks:

- **1.1 Chat Input Reception & Initial Handling:** Receives chat input from the website widget and determines if input exists.
- **1.2 AI Agent Processing:** Uses OpenAI GPT-4 to interpret user input, manage conversation context, and decide next actions.
- **1.3 Calendar Availability & Free Slot Calculation:** Queries Microsoft Outlook calendar events, processes busy times, and calculates free time slots within business hours.
- **1.4 Appointment Booking:** Books appointments in Outlook calendar based on user preferences and availability.
- **1.5 Human Handoff via Email:** Sends detailed enquiry emails to a human contact if the user is not ready to book or has questions outside the bot’s remit.
- **1.6 Response Delivery:** Sends chatbot responses back to the website widget.

Supporting nodes include memory buffers for conversation context, workflow triggers, and utility nodes for data formatting.

---

### 2. Block-by-Block Analysis

#### 1.1 Chat Input Reception & Initial Handling

- **Overview:**  
  This block receives incoming chat messages from the website widget via webhook and checks if the user input exists to proceed or respond with a default greeting.

- **Nodes Involved:**  
  - Chat Trigger (disabled)  
  - If  
  - Respond With Initial Message

- **Node Details:**  

  - **Chat Trigger**  
    - Type: Chat Trigger (LangChain)  
    - Role: Entry point webhook for chat messages from the website widget.  
    - Config: Public webhook, response mode set to respond via node, allows all origins.  
    - Disabled: Yes (likely replaced by Execute Workflow Trigger node).  
    - Edge Cases: Disabled node will not trigger; ensure webhook is enabled or replaced.

  - **If**  
    - Type: Conditional node  
    - Role: Checks if the incoming JSON contains a non-empty `chatInput` field.  
    - Config: Condition checks existence of `$json.chatInput`.  
    - Inputs: From Chat Trigger or Execute Workflow Trigger.  
    - Outputs:  
      - True: Proceed to AI Agent processing.  
      - False: Respond with initial greeting.  
    - Edge Cases: Missing or malformed input JSON may cause false negatives.

  - **Respond With Initial Message**  
    - Type: Respond to Webhook  
    - Role: Sends a default greeting JSON response: `{"output": "Hi, how can I help you today?"}`  
    - Inputs: From If node (false branch).  
    - Outputs: Response sent back to chat widget.  
    - Edge Cases: If webhook context lost, response may fail.

---

#### 1.2 AI Agent Processing

- **Overview:**  
  This block uses an AI agent powered by OpenAI GPT-4 to interpret user messages, maintain conversation context, and decide whether to check availability, book appointments, or send messages to a human.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Window Buffer Memory  
  - Respond to Webhook

- **Node Details:**  

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Core conversational AI that processes user input and orchestrates calls to tools/workflows.  
    - Config:  
      - Input text from `$json.chatInput`.  
      - System message defines assistant role as an executive PA for appointment coordination with detailed instructions and constraints (e.g., UK timezone, business hours, no double booking, fallback to human email).  
      - Uses sub-workflows as tools for availability checking, appointment booking, and sending messages.  
    - Inputs: From If node (true branch).  
    - Outputs: To Respond to Webhook node.  
    - Edge Cases:  
      - OpenAI API errors (rate limits, auth failures).  
      - Logic errors if placeholders not replaced.  
      - Timezone or date parsing errors.  
      - If calendar API unavailable, must notify user.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4 language model responses for the AI Agent.  
    - Config: Model set to `gpt-4o-2024-08-06` with temperature 0.4 for balanced creativity.  
    - Credentials: OpenAI API key configured.  
    - Inputs: From AI Agent (as language model).  
    - Outputs: To AI Agent.  
    - Edge Cases: API quota exhaustion, network timeouts.

  - **Window Buffer Memory**  
    - Type: LangChain Memory Buffer (Window)  
    - Role: Maintains conversation history limited to last 20 messages per session (keyed by sessionId).  
    - Config: Session key from `$json.sessionId`.  
    - Inputs: From AI Agent (memory).  
    - Outputs: To AI Agent.  
    - Edge Cases: Session key missing or malformed.

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends AI Agent’s response back to the chat widget.  
    - Inputs: From AI Agent.  
    - Outputs: HTTP response to chat widget.  
    - Edge Cases: Webhook context lost, response timeout.

---

#### 1.3 Calendar Availability & Free Slot Calculation

- **Overview:**  
  This block retrieves calendar events from Microsoft Outlook, processes busy events, and calculates free time slots within defined business hours for the next two weeks.

- **Nodes Involved:**  
  - Execute Workflow Trigger  
  - Switch  
  - Get Events  
  - freeTimeSlots (Code)  
  - varResponse  
  - Get Availability (sub-workflow)

- **Node Details:**  

  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for external triggers (e.g., chat commands) to start availability or message workflows.  
    - Inputs: External trigger with query parameters.  
    - Outputs: To Switch node.  
    - Edge Cases: Trigger misconfiguration.

  - **Switch**  
    - Type: Switch node  
    - Role: Routes workflow based on `route` field in JSON (`availability` or `message`).  
    - Inputs: From Execute Workflow Trigger.  
    - Outputs:  
      - `availability` → Get Events  
      - `message` → Send Message1 (email)  
    - Edge Cases: Unknown route values.

  - **Get Events**  
    - Type: HTTP Request  
    - Role: Calls Microsoft Graph API to fetch calendar events for next 14 days (from 2 days ahead to 16 days ahead).  
    - Config:  
      - URL: `https://graph.microsoft.com/v1.0/me/calendarView`  
      - Query parameters: startDateTime, endDateTime, select fields, order by start time ascending.  
      - Header: Prefer timezone Europe/London.  
      - Auth: Microsoft Outlook OAuth2.  
    - Inputs: From Switch node (availability branch).  
    - Outputs: To freeTimeSlots node.  
    - Edge Cases: OAuth token expiry, API rate limits, network errors.

  - **freeTimeSlots**  
    - Type: Code (JavaScript)  
    - Role: Processes events to identify free time slots within business hours (08:00-17:30 UK time) Monday to Friday, excluding weekends.  
    - Logic:  
      - Parses events by date, sorts busy events, finds gaps between busy times.  
      - Outputs array of free slots with start/end ISO strings and day of week.  
    - Inputs: From Get Events.  
    - Outputs: To varResponse.  
    - Edge Cases: Empty event list, malformed dates, timezone inconsistencies.

  - **varResponse**  
    - Type: Set node  
    - Role: Converts freeTimeSlots array to JSON string in `response` field for downstream use.  
    - Inputs: From freeTimeSlots.  
    - Outputs: To AI Agent (via Get Availability sub-workflow).  
    - Edge Cases: JSON serialization errors.

  - **Get Availability**  
    - Type: Tool Workflow (sub-workflow)  
    - Role: Invoked by AI Agent to check calendar availability before booking. Returns all events for next 2 weeks.  
    - Inputs: None directly; triggered by AI Agent with route `availability`.  
    - Outputs: Events data for processing.  
    - Edge Cases: Sub-workflow failures propagate to AI Agent.

---

#### 1.4 Appointment Booking

- **Overview:**  
  This block books a 30-minute appointment in Microsoft Outlook calendar based on user details and selected time slot.

- **Nodes Involved:**  
  - Make Appointment (HTTP Request)  
  - AI Agent (tool invocation)

- **Node Details:**  

  - **Make Appointment**  
    - Type: HTTP Request (LangChain toolHttpRequest)  
    - Role: Sends POST request to Microsoft Graph API to create an event (appointment).  
    - Config:  
      - URL: `https://graph.microsoft.com/v1.0/me/events`  
      - Method: POST  
      - JSON body with placeholders for subject, start/end datetime (ISO with timezone Europe/London), body content, attendees, online meeting details (Teams), categories, and status.  
      - Auth: Microsoft Outlook OAuth2.  
      - Headers: Content-Type application/json.  
      - Placeholders: dateStartTime, dateEndTime, reason, email, name.  
    - Inputs: From AI Agent tool invocation with filled parameters.  
    - Outputs: To AI Agent.  
    - Edge Cases: Auth token expiry, invalid date formats, API errors, overlapping appointments.

---

#### 1.5 Human Handoff via Email

- **Overview:**  
  This block sends detailed enquiry emails to a human contact when the user is not ready to book or has questions outside the bot’s remit.

- **Nodes Involved:**  
  - Send Message (Tool Workflow)  
  - Send Message1 (Microsoft Outlook node)  
  - varMessageResponse (Set node)  
  - AI Agent (tool invocation)

- **Node Details:**  

  - **Send Message**  
    - Type: Tool Workflow (sub-workflow)  
    - Role: Invoked by AI Agent to send an email message to a human contact.  
    - Inputs: JSON schema requiring email, subject, message, name, company.  
    - Outputs: To AI Agent.  
    - Edge Cases: Sub-workflow failures.

  - **Send Message1**  
    - Type: Microsoft Outlook node  
    - Role: Sends formatted HTML email to a predefined recipient (e.g., founder’s email).  
    - Config:  
      - Subject and body content dynamically populated from Execute Workflow Trigger query parameters.  
      - Importance set to High.  
      - HTML email template includes customer details and message.  
      - Recipient email hardcoded as `you@yourdomain.com` (to be customized).  
      - Auth: Microsoft Outlook OAuth2.  
    - Inputs: From Switch node (message branch).  
    - Outputs: To varMessageResponse.  
    - Edge Cases: Email send failures, invalid recipient address.

  - **varMessageResponse**  
    - Type: Set node  
    - Role: Stores email send response in `response` field.  
    - Inputs: From Send Message1.  
    - Outputs: To AI Agent (via tool invocation).  
    - Edge Cases: None significant.

---

#### 1.6 Response Delivery

- **Overview:**  
  Sends final chatbot responses back to the website widget after AI processing or email sending.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends the AI Agent’s final response JSON back to the chat widget.  
    - Inputs: From AI Agent.  
    - Outputs: HTTP response to chat widget.  
    - Edge Cases: Webhook context lost, response timeout.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                  | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                   |
|-------------------------|----------------------------------|-------------------------------------------------|---------------------------|---------------------------|-----------------------------------------------------------------------------------------------|
| Chat Trigger            | LangChain Chat Trigger            | Entry webhook for chat messages                  | —                         | If                        | Disabled node; replaced by Execute Workflow Trigger                                           |
| If                      | Conditional                      | Checks if chat input exists                       | Chat Trigger / Execute Workflow Trigger | AI Agent / Respond With Initial Message |                                                                                               |
| Respond With Initial Message | Respond to Webhook               | Sends default greeting if no input               | If                        | —                         |                                                                                               |
| AI Agent                | LangChain Agent                  | Core AI conversational agent                      | If (true branch)          | Respond to Webhook         | System prompt defines PA role, UK timezone, appointment rules, fallback email                  |
| OpenAI Chat Model       | LangChain OpenAI Chat Model      | GPT-4 language model for AI Agent                 | AI Agent (languageModel)  | AI Agent                  |                                                                                               |
| Window Buffer Memory    | LangChain Memory Buffer (Window) | Maintains conversation context                    | AI Agent (memory)         | AI Agent                  |                                                                                               |
| Respond to Webhook      | Respond to Webhook               | Sends AI response back to chat widget             | AI Agent                  | —                         |                                                                                               |
| Execute Workflow Trigger | Execute Workflow Trigger          | External trigger entry point                       | —                         | Switch                    |                                                                                               |
| Switch                  | Switch                          | Routes workflow by `route` field                   | Execute Workflow Trigger  | Get Events / Send Message1 |                                                                                               |
| Get Events              | HTTP Request                    | Fetches calendar events from Microsoft Graph      | Switch (availability)     | freeTimeSlots             |                                                                                               |
| freeTimeSlots           | Code                           | Calculates free time slots within business hours  | Get Events                | varResponse               |                                                                                               |
| varResponse             | Set                            | Converts freeTimeSlots array to JSON string       | freeTimeSlots             | AI Agent (via Get Availability) | Sticky Note: "Ensure these reference this workflow, replace placeholders"                      |
| Get Availability        | Tool Workflow (sub-workflow)    | Sub-workflow to check calendar availability       | AI Agent (tool)           | Get Events                |                                                                                               |
| Make Appointment        | HTTP Request (toolHttpRequest)  | Books appointment in Outlook calendar             | AI Agent (tool)           | AI Agent                  | Sticky Note: "Ensure these reference this workflow, replace placeholders"                      |
| Send Message            | Tool Workflow (sub-workflow)    | Sub-workflow to send enquiry email to human       | AI Agent (tool)           | AI Agent                  |                                                                                               |
| Send Message1           | Microsoft Outlook               | Sends formatted enquiry email                      | Switch (message)          | varMessageResponse        | Sticky Note: "Customise the email template"                                                   |
| varMessageResponse      | Set                            | Stores email send response                          | Send Message1             | AI Agent                  |                                                                                               |
| Sticky Note             | Sticky Note                    | Instructional note                                 | —                         | —                         | "Ensure these reference this workflow, replace placeholders"                                  |
| Sticky Note1            | Sticky Note                    | Instructional note                                 | —                         | —                         | "modify business hours\nmodify timezones"                                                    |
| Sticky Note2            | Sticky Note                    | Branding and setup blog link                        | —                         | —                         | "Read to blog post to get started 📝\nFollow along to add a custom branded chat widget..."    |
| Sticky Note3            | Sticky Note                    | Branding and video link                             | —                         | —                         |                                                                                               |
| Sticky Note4            | Sticky Note                    | Instructional note                                 | —                         | —                         | "modify timezones"                                                                            |
| Sticky Note8            | Sticky Note                    | Branding and credits                               | —                         | —                         | "Custom Branded n8n Chatbot\nBuilt by Wayne Simpson at nocodecreative.io\nBuy me a coffee"    |
| Sticky Note9            | Sticky Note                    | Setup video link                                   | —                         | —                         | "Watch the Setup Video 📺\nWatch Set Up Video 👇\n[YouTube link]"                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Input Reception:**
   - Add a **Webhook** node or **Chat Trigger** (LangChain) node configured as a public webhook to receive chat messages from the website widget.
   - Set response mode to respond via node.
   - Connect to an **If** node.

2. **Add If Node to Check Input:**
   - Create an **If** node.
   - Condition: Check if `$json.chatInput` exists and is not empty.
   - True branch connects to AI Agent block.
   - False branch connects to a **Respond to Webhook** node sending JSON: `{"output": "Hi, how can I help you today?"}`.

3. **Configure AI Agent Block:**
   - Add **AI Agent** node (LangChain Agent).
   - Input text: `={{ $json.chatInput }}`.
   - System message: Use detailed prompt defining assistant role as executive PA, UK timezone, appointment rules, fallback email instructions (copy from original).
   - Connect AI Agent’s language model input to an **OpenAI Chat Model** node:
     - Model: `gpt-4o-2024-08-06`
     - Temperature: 0.4
     - Credentials: OpenAI API key.
   - Connect AI Agent’s memory input/output to a **Window Buffer Memory** node:
     - Session key: `={{ $json.sessionId }}`
     - Context window length: 20.
   - Connect AI Agent output to a **Respond to Webhook** node to send responses back.

4. **Create Execute Workflow Trigger:**
   - Add **Execute Workflow Trigger** node as an external entry point for commands like availability or message.
   - Connect to a **Switch** node.

5. **Configure Switch Node:**
   - Add **Switch** node.
   - Condition on `$json.route`:
     - If equals `availability`, route to Get Events node.
     - If equals `message`, route to Send Message1 node.

6. **Setup Calendar Event Retrieval:**
   - Add **HTTP Request** node named Get Events.
   - URL: `https://graph.microsoft.com/v1.0/me/calendarView`
   - Query parameters:
     - `startDateTime`: `={{ new Date(new Date().setDate(new Date().getDate() + 2)).toISOString() }}`
     - `endDateTime`: `={{ new Date(new Date().setDate(new Date().getDate() + 16)).toISOString() }}`
     - `$top`: 50
     - `select`: `start,end,categories,importance,isAllDay,recurrence,showAs,subject,type`
     - `orderby`: `start/dateTime asc`
   - Header: `Prefer: outlook.timezone="Europe/London"`
   - Authentication: Microsoft Outlook OAuth2 credentials.

7. **Add freeTimeSlots Code Node:**
   - Add **Code** node named freeTimeSlots.
   - Paste JavaScript code that:
     - Parses events.
     - Defines business hours 08:00-17:30 UK time.
     - Finds free slots excluding weekends.
     - Outputs freeTimeSlots array with date, dayOfWeek, freeStart, freeEnd, timeZone.
   - Connect Get Events output to this node.

8. **Add varResponse Set Node:**
   - Add **Set** node named varResponse.
   - Assign `response` field as JSON string of freeTimeSlots: `={{ $json.freeTimeSlots.toJsonString() }}`.
   - Connect freeTimeSlots output here.

9. **Create Get Availability Sub-Workflow:**
   - Create a sub-workflow named "Get_availability".
   - It should return calendar events for next 2 weeks.
   - Configure AI Agent to call this sub-workflow as a tool with route `availability`.

10. **Configure Appointment Booking:**
    - Add **HTTP Request** node named Make Appointment.
    - URL: `https://graph.microsoft.com/v1.0/me/events`
    - Method: POST
    - JSON body with placeholders for subject, start/end datetime (ISO string Europe/London), body content, attendees, online meeting details (Teams), categories, showAs busy.
    - Headers: Content-Type application/json.
    - Authentication: Microsoft Outlook OAuth2.
    - Connect AI Agent to invoke this node as a tool.

11. **Setup Human Handoff Email:**
    - Create sub-workflow "Send_email" to send enquiry emails.
    - Add **Microsoft Outlook** node named Send Message1.
    - Configure with dynamic subject and HTML body content using Execute Workflow Trigger query parameters.
    - Recipient: set to your human contact email.
    - Importance: High.
    - Authentication: Microsoft Outlook OAuth2.
    - Connect Switch node (message branch) to Send Message1.
    - Add **Set** node varMessageResponse to store response.
    - Connect Send Message1 output to varMessageResponse.

12. **Connect AI Agent to Send Message Sub-Workflow:**
    - Configure AI Agent to invoke Send_email sub-workflow as a tool with route `message`.

13. **Add Sticky Notes:**
    - Add sticky notes with instructions on replacing placeholders, modifying business hours/timezones, customizing email templates, and branding credits.

14. **Configure Credentials:**
    - Setup OpenAI API credentials.
    - Setup Microsoft Outlook OAuth2 credentials with appropriate scopes for calendar and mail.

15. **Add JavaScript Snippet to Website:**
    - Add provided JavaScript snippet (from blog) to your website to connect chat widget to the webhook URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Widget includes a "Powered By" affiliate link                                                                 | Visible in chat widget UI                                                                               |
| Follow our detailed setup guide to get started in minutes                                                     | https://blog.nocodecreative.io/create-a-branded-ai-powered-website-chatbot-with-n8n/                    |
| Setup video available on YouTube                                                                              | https://youtu.be/xQ1tCQZhLaI                                                                            |
| Built by Wayne Simpson at nocodecreative.io                                                                   | https://nocodecreative.io and https://www.linkedin.com/in/simpsonwayne/                                 |
| Coffee support link                                                                                            | https://ko-fi.com/waynesimpson                                                                          |
| Business hours and timezone are critical: 8am-6pm Monday-Friday Europe/London timezone                         | Modify in code node and AI Agent system prompt as needed                                               |
| All appointments require minimum 48 hours' notice and are 30 minutes long                                      | Enforced by AI Agent logic                                                                              |
| Always confirm appointment details with customer and send calendar invite                                     | AI Agent instructions                                                                                   |
| If unable to book appointment (e.g., calendar API down), inform customer politely                              | AI Agent fallback behavior                                                                              |
| Email template is customizable in Send Message1 node                                                          | Modify HTML content and recipient email address                                                        |

---

This structured documentation enables understanding, modification, and reproduction of the entire branded AI chatbot workflow with calendar integration and human fallback.