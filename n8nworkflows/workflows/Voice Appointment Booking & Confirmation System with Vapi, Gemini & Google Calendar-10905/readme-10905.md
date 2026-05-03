Voice Appointment Booking & Confirmation System with Vapi, Gemini & Google Calendar

https://n8nworkflows.xyz/workflows/voice-appointment-booking---confirmation-system-with-vapi--gemini---google-calendar-10905


# Voice Appointment Booking & Confirmation System with Vapi, Gemini & Google Calendar

---
### 1. Workflow Overview

This workflow enables an AI voice-based appointment booking and confirmation system integrating Vapi AI assistant, Google Calendar, and Gmail. It serves two primary use cases:

- **Availability Checking:** When a user queries about available appointment slots for a specific day, the system fetches existing calendar events, formats busy periods, and uses AI to analyze and return available time slots.
- **Appointment Booking and Confirmation:** When an appointment is confirmed by the user, the system creates a corresponding Google Calendar event, sends a professional confirmation email to the client, and returns a confirmation response to the AI assistant.

The workflow is logically divided into two main blocks:

- **1.1 Availability Checker Tool:** Receives availability requests, queries calendar, processes busy slots, analyzes availability via AI, and responds to Vapi.
- **1.2 Appointment Creator Tool:** Receives appointment details, creates calendar event, sends confirmation email, and responds with booking confirmation.

Supporting nodes include labeling sticky notes for clarity and setup instructions for deployment.

---

### 2. Block-by-Block Analysis

#### 2.1 Availability Checker Tool

**Overview:**  
This block handles incoming requests from Vapi asking for available booking times on a given date. It queries Google Calendar for busy slots, formats the data, uses an AI agent to analyze availability within business hours, and returns the available slots back to Vapi.

**Nodes Involved:**  
- Webhook - Availability Checker  
- Get Busy Slots from Calendar  
- Format Busy Slots  
- Google Gemini Chat Model  
- AI Availability Analyzer  
- Return Availability to Vapi

**Node Details:**

- **Webhook - Availability Checker**  
  - Type: Webhook (HTTP POST endpoint)  
  - Role: Entry point for availability check requests from Vapi assistant.  
  - Configuration: HTTP POST at path `/availability-checker`, response mode set to respond from downstream node output.  
  - Key Variables: Receives JSON body containing date parameter under `message.toolCalls[0].function.arguments.date`.  
  - Connections: Output to "Get Busy Slots from Calendar".  
  - Edge Cases: Invalid or missing date parameter; HTTP errors; webhook not activated.

- **Get Busy Slots from Calendar**  
  - Type: Google Calendar node  
  - Role: Retrieves busy time slots from the selected Google Calendar for the requested date.  
  - Configuration:  
    - Calendar selected from user’s connected Google account.  
    - Time range constrained to start and end of requested date, converted via internal toDateTime operations.  
    - Timezone set to America/New_York by default.  
    - Output format: raw calendar data including busy slots.  
  - Input: JSON body from webhook containing date info.  
  - Output: Calendar busy slots data to "Format Busy Slots".  
  - Edge Cases: Authorization errors with Google API; invalid calendar; empty calendar; timezone mismatch.

- **Format Busy Slots**  
  - Type: Code node (JavaScript)  
  - Role: Parses raw busy slot data, formats it into human-readable text lines for AI consumption.  
  - Configuration: Custom JS code extracts busy slots array and maps each slot to a formatted string line with start and end time. Returns JSON with plain text `busySlots` and raw data `rawBusy`.  
  - Input: Google Calendar busy slots JSON.  
  - Output: Formatted availability text to AI agent.  
  - Edge Cases: Missing or empty busy slots; unexpected calendar data structure.

- **Google Gemini Chat Model**  
  - Type: AI language model node (Google Gemini)  
  - Role: Provides AI language model capabilities as backend for the AI agent node.  
  - Configuration: Default options, no special parameters.  
  - Input: Connected as AI LLM provider for the "AI Availability Analyzer" node.  
  - Output: N/A (used internally by agent).  
  - Edge Cases: API quota exceeded, authentication failure, latency.

- **AI Availability Analyzer**  
  - Type: Langchain Agent node (AI agent)  
  - Role: Analyzes busy slots text and requested date, returns available time slots within business hours (9am-6pm).  
  - Configuration:  
    - Prompt includes busy slots and date variables.  
    - System message instructs the agent to only return available times in plain language.  
    - Language model set to Google Gemini via linked node.  
  - Input: Formatted busy slots from previous node, date from webhook.  
  - Output: Plain text availability result to "Return Availability to Vapi".  
  - Edge Cases: AI model misinterpretation, empty or ambiguous input, network/API errors.

- **Return Availability to Vapi**  
  - Type: Respond to Webhook  
  - Role: Sends back the AI-analyzed availability results as JSON response to Vapi.  
  - Configuration:  
    - Responds with JSON formatted to Vapi's required schema, including `toolCallId` and availability result string.  
  - Input: AI analyzed availability text.  
  - Output: HTTP response to webhook caller.  
  - Edge Cases: Response formatting errors, webhook connection issues.

---

#### 2.2 Appointment Creator Tool

**Overview:**  
This block processes confirmed appointment details received from Vapi, creates a Google Calendar event accordingly, sends a confirmation email to the customer, and returns a confirmation response to Vapi.

**Nodes Involved:**  
- Webhook - Create Appointment  
- Create Calendar Event  
- Send Confirmation Email  
- Return Confirmation to Vapi

**Node Details:**

- **Webhook - Create Appointment**  
  - Type: Webhook (HTTP POST endpoint)  
  - Role: Entry point for appointment creation requests from Vapi.  
  - Configuration: HTTP POST at path `/create-appointment`, response mode set to respond from downstream node output.  
  - Key Variables: Extracts appointment details such as Name, Email, Appointment type, and date/time from JSON body under `message.toolCalls[0].function.arguments`.  
  - Connections: Output to "Create Calendar Event".  
  - Edge Cases: Missing required parameters; invalid date/time format; webhook inactive.

- **Create Calendar Event**  
  - Type: Google Calendar node  
  - Role: Creates a calendar event in the selected Google Calendar based on appointment details.  
  - Configuration:  
    - Calendar selected from user’s connected Google account.  
    - Event start time from appointment date/time argument.  
    - Event end time is 30 minutes after start by default (configurable).  
    - Event summary composed dynamically: “[Appointment type] Appointment for [Name]”.  
    - Description includes appointment type and email.  
  - Input: Appointment details from webhook.  
  - Output: Created event data to "Send Confirmation Email".  
  - Edge Cases: Google API quota limits; invalid calendar; overlapping events; incorrect timezones.

- **Send Confirmation Email**  
  - Type: Gmail node  
  - Role: Sends a professional HTML confirmation email to the customer confirming the appointment.  
  - Configuration:  
    - Recipient email dynamically set from appointment Email argument.  
    - Subject: “Appointment Confirmation”.  
    - HTML email body dynamically inserts business name, appointment type, date/time formatted, and customer name.  
    - Gmail credentials must be configured for sending.  
  - Input: Created calendar event data and appointment details.  
  - Output: Confirmation email sent, passes data to "Return Confirmation to Vapi".  
  - Edge Cases: Gmail authentication failures; invalid email addresses; sending limits; HTML rendering issues.

- **Return Confirmation to Vapi**  
  - Type: Respond to Webhook  
  - Role: Sends a JSON confirmation response back to Vapi indicating success and recipient email.  
  - Configuration:  
    - Response JSON includes toolCallId and a success message referencing the customer email.  
  - Input: Confirmation email node output.  
  - Output: HTTP response to webhook caller.  
  - Edge Cases: Response formatting or delivery errors.

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                                   | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                                             |
|-----------------------------|-----------------------------|-------------------------------------------------|-------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Webhook - Availability Checker | Webhook                    | Receives availability check requests from Vapi  |                               | Get Busy Slots from Calendar  | ## Availability Checker Tool This section checks calendar availability and returns open time slots to Vapi.                            |
| Get Busy Slots from Calendar | Google Calendar             | Fetches busy calendar slots for requested date   | Webhook - Availability Checker | Format Busy Slots             |                                                                                                                                         |
| Format Busy Slots            | Code                       | Formats busy slots into human-readable text      | Get Busy Slots from Calendar   | AI Availability Analyzer      |                                                                                                                                         |
| Google Gemini Chat Model     | AI Language Model (Gemini) | Provides AI LLM backend for availability analysis |                               | AI Availability Analyzer (as model) |                                                                                                                                         |
| AI Availability Analyzer     | Langchain Agent            | Analyzes busy slots and returns available times  | Format Busy Slots              | Return Availability to Vapi   |                                                                                                                                         |
| Return Availability to Vapi  | Respond to Webhook          | Returns available time slots to Vapi              | AI Availability Analyzer       |                              |                                                                                                                                         |
| Webhook - Create Appointment | Webhook                    | Receives appointment creation requests from Vapi |                               | Create Calendar Event         | ## Appointment Creator Tool This section creates the calendar event and sends confirmation email to the customer.                     |
| Create Calendar Event        | Google Calendar             | Creates calendar event based on appointment data | Webhook - Create Appointment   | Send Confirmation Email       |                                                                                                                                         |
| Send Confirmation Email      | Gmail                      | Sends confirmation email to customer              | Create Calendar Event          | Return Confirmation to Vapi   |                                                                                                                                         |
| Return Confirmation to Vapi  | Respond to Webhook          | Returns appointment confirmation to Vapi          | Send Confirmation Email        |                              |                                                                                                                                         |
| Setup Instructions           | Sticky Note                | Provides detailed setup and configuration guide   |                               |                              | ## AI Voice Appointment Booking with Vapi and Google Calendar ... (full content describing setup steps, customization, and requirements) |
| Availability Section Label   | Sticky Note                | Labels the availability checker section           |                               |                              | ## Availability Checker Tool This section checks calendar availability and returns open time slots to Vapi.                            |
| Booking Section Label        | Sticky Note                | Labels the appointment creation section            |                               |                              | ## Appointment Creator Tool This section creates the calendar event and sends confirmation email to the customer.                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook - Availability Checker**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/availability-checker`  
   - Response Mode: Response Node  
   - Purpose: Receive availability check requests from Vapi.

2. **Create Google Calendar Node - Get Busy Slots from Calendar**  
   - Connect Google Calendar account with OAuth2 credentials.  
   - Calendar: Select your booking calendar.  
   - Timezone: Set to your local timezone (default: America/New_York).  
   - TimeMin: Expression - `{{$json.body.message.toolCalls[0].function.arguments.date.toDateTime().startOf('day')}}`  
   - TimeMax: Expression - `{{$json.body.message.toolCalls[0].function.arguments.date.toDateTime().endOf('day')}}`  
   - Resource: `calendar`  
   - Connect output from "Webhook - Availability Checker".

3. **Create Code Node - Format Busy Slots**  
   - Paste the following JavaScript code to format busy slots:  
     ```js
     const calendarId = Object.keys($json.calendars)[0];
     const busySlots = $json.calendars[calendarId].busy || [];
     const formattedSlots = busySlots.map(slot => `• ${slot.start} → ${slot.end}`).join("\n");
     return [{ json: { busySlots: formattedSlots, rawBusy: busySlots } }];
     ```  
   - Connect output from "Get Busy Slots from Calendar".

4. **Create Google Gemini Chat Model Node**  
   - Type: AI language model node for Google Gemini (requires API key and access).  
   - Default options.

5. **Create Langchain Agent Node - AI Availability Analyzer**  
   - Connect AI language model input to "Google Gemini Chat Model".  
   - Text parameter:  
     ```
     These are the busy slots of the day: {{$json.busySlots}}
     Here is the day for which the user is looking to book: {{$('Webhook - Availability Checker').item.json.body.message.toolCalls[0].function.arguments.date}}
     ```  
   - System Message:  
     ```
     You are a helpful assistant. Your job is to take the busy/booked time slots of the day and return the available time slots for booking. We are open from 9:00am to 6:00pm. You just return the available times in plain language and nothing else.
     
     Example: "Available from 9:00am to 1:00pm and 3:00pm to 6:00pm"
     ```  
   - Connect output from "Format Busy Slots".

6. **Create Respond to Webhook Node - Return Availability to Vapi**  
   - Respond With: JSON  
   - Response Body (Expression):  
     ```json
     {
       "results": [
         {
           "toolCallId": "={{ $('Webhook - Availability Checker').item.json.body.message.toolCalls[0].id }}",
           "result": "={{ $json.output }}"
         }
       ]
     }
     ```  
   - Connect output from "AI Availability Analyzer".

7. **Create Webhook - Create Appointment**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/create-appointment`  
   - Response Mode: Response Node  
   - Purpose: Receive appointment confirmation details from Vapi.

8. **Create Google Calendar Node - Create Calendar Event**  
   - Connect Google Calendar account.  
   - Calendar: Select the same booking calendar as before.  
   - Start: Expression - `={{ $json.body.message.toolCalls[0].function.arguments["date and time"].toDateTime() }}`  
   - End: Expression - `={{ $json.body.message.toolCalls[0].function.arguments["date and time"].toDateTime().plus(30, 'mins') }}`  
   - Additional Fields:  
     - Summary: `={{ $json.body.message.toolCalls[0].function.arguments["Appointment type"] + " Appointment for " + $json.body.message.toolCalls[0].function.arguments.Name }}`  
     - Description:  
       ```
       Appointment type: {{$json.body.message.toolCalls[0].function.arguments["Appointment type"]}}
       Email: {{$json.body.message.toolCalls[0].function.arguments.Email}}
       ```  
   - Connect output from "Webhook - Create Appointment".

9. **Create Gmail Node - Send Confirmation Email**  
   - Connect Gmail account with OAuth2 credentials.  
   - To: Expression - `={{ $('Webhook - Create Appointment').item.json.body.message.toolCalls[0].function.arguments.Email }}`  
   - Subject: `Appointment Confirmation`  
   - Message: Use the provided HTML email template with dynamic placeholders for customer name, appointment type, and formatted date/time.  
   - Options: Disable attribution.  
   - Connect output from "Create Calendar Event".

10. **Create Respond to Webhook Node - Return Confirmation to Vapi**  
    - Respond With: JSON  
    - Response Body (Expression):  
      ```json
      {
        "results": [
          {
            "toolCallId": "={{ $('Webhook - Create Appointment').item.json.body.message.toolCallList[0].id }}",
            "result": "Appointment confirmed and confirmation email sent to {{ $('Webhook - Create Appointment').item.json.body.message.toolCalls[0].function.arguments.Email }}"
          }
        ]
      }
      ```  
    - Connect output from "Send Confirmation Email".

11. **Add Sticky Notes for Documentation**  
    - Add notes describing each section and setup instructions as described in the Setup Instructions node content.  
    - Include instructions for configuring Vapi tools to point to the two webhook URLs, setting Google Calendar and Gmail credentials, and updating business hours in the AI agent prompt.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| This workflow integrates Vapi AI voice assistant with Google Calendar and Gmail to automate appointment booking and confirmation via voice commands. It requires properly configured Vapi tools to call the provided webhook endpoints. Google Calendar API and Gmail API credentials must be set up with OAuth2 and appropriate scopes. Business hours and appointment duration can be customized in the AI agent prompt and calendar event node respectively. The confirmation email template is fully customizable in HTML to match branding. | Workflow description and setup overview            |
| Official n8n documentation for webhook usage: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | n8n Webhook Node Documentation                      |
| Google Calendar API guide for event management: https://developers.google.com/calendar/api/guides/events                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Google Calendar API Documentation                   |
| Gmail node configuration and limitations: https://docs.n8n.io/nodes/n8n-nodes-base.gmail/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | n8n Gmail Node Documentation                         |
| Google Gemini and Langchain integration requires API access and may be subject to quota and latency considerations. Ensure API keys are securely configured.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Google Gemini & Langchain integration notes         |

---

**Disclaimer:**  
The provided content originates exclusively from a workflow automated using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.