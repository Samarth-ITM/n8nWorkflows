Get Daily Calendar Summaries via SMS with Google Calendar, Twilio and Claude AI

https://n8nworkflows.xyz/workflows/get-daily-calendar-summaries-via-sms-with-google-calendar--twilio-and-claude-ai-10134


# Get Daily Calendar Summaries via SMS with Google Calendar, Twilio and Claude AI

### 1. Workflow Overview

This workflow automates the process of sending a concise daily SMS summary of a user's calendar events fetched from Google Calendar. Scheduled to run every day at 7 AM, it retrieves all events for the current day, formats them into a friendly and informative message using an AI language model, and sends the SMS via Twilio. The workflow is designed for personal or family use to provide a helpful morning reminder of the day's agenda.

Logical blocks:
- **1.1 Scheduled Trigger:** Initiates the workflow daily at 7 AM.
- **1.2 Google Calendar Integration:** Retrieves all calendar events for the current day.
- **1.3 Event Presence Check:** Determines if there are any events to summarize.
- **1.4 Event Formatting:** Converts raw event data into a structured textual list and associates the recipient phone number.
- **1.5 AI Summarization:** Uses an AI language model to generate a friendly, concise SMS summary.
- **1.6 SMS Sending:** Sends the generated summary via Twilio SMS service.
- **1.7 Supporting Documentation:** Sticky notes provide user guidance and contextual information.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Triggers the workflow automatically every day at 7 AM to ensure timely delivery of the daily calendar summary.
- **Nodes Involved:**  
  - `Trigger workflow at 7AM`
- **Node Details:**

  - **Trigger workflow at 7AM**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger at 7:00 AM daily using the interval triggerAtHour parameter.  
    - Inputs: None (start node)  
    - Outputs: Connects to "Get events from Events Calendar" node  
    - Edge Cases: Workflow may fail if the timezone setting is incorrect or if the server is down at trigger time.  
    - Version: 1.2  

#### 1.2 Google Calendar Integration

- **Overview:** Fetches all events from the specified Google Calendar for the current day (from midnight to midnight).
- **Nodes Involved:**  
  - `Get events from Events Calendar`
- **Node Details:**

  - **Get events from Events Calendar**  
    - Type: Google Calendar  
    - Configuration:  
      - Operation: getAll events  
      - Time range: From start of today (`$today`) to start of tomorrow (`$today.plus(1, 'day')`)  
      - Calendar: Configurable list selection (empty by default, user must specify calendar)  
      - Return all events (no limit)  
    - Inputs: Receives trigger from schedule node  
    - Outputs: Passes event data to "Check if there are any events" node  
    - Credentials: Requires Google Calendar OAuth2  
    - Edge Cases: Possible authentication errors; empty event list if no events scheduled; calendar not selected or misconfigured.  
    - Version: 1.3  

#### 1.3 Event Presence Check

- **Overview:** Checks whether any events were returned by the calendar query to decide if further processing is needed.
- **Nodes Involved:**  
  - `Check if there are any events`
- **Node Details:**

  - **Check if there are any events**  
    - Type: If Condition  
    - Configuration:  
      - Condition: Number of items from previous node > 0  
      - Logic: If true, proceed; if false, workflow ends or no further action is taken.  
    - Inputs: Receives event list from Google Calendar node  
    - Outputs:  
      - True branch: To "Format list of events and sender mobile num" node  
      - False branch: No continuation (no SMS sent if no events)  
    - Edge Cases: Condition must be strictly numeric; expression failures if input is malformed.  
    - Version: 2.2  

#### 1.4 Event Formatting

- **Overview:** Formats the list of events into a user-friendly string, including times and event summaries, and sets the recipient phone number for SMS.
- **Nodes Involved:**  
  - `Format list of events and sender mobile num`
- **Node Details:**

  - **Format list of events and sender mobile num**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Custom JS code iterates over all events, extracts summary and start time (localized), formats each event as a bullet point with time.  
      - Outputs an object with:  
        - `to`: Phone number string (hardcoded as "+12334567890" and should be replaced per user)  
        - `eventsList`: Concatenated event list string or "No events today."  
        - `calendar`: User name or calendar identifier (hardcoded as "Anne")  
    - Inputs: Receives event data from the If node's true branch  
    - Outputs: Passes formatted data to "Persist schema" node  
    - Edge Cases: Timezone or date parsing errors if event data malformed; hardcoded phone number must be customized; empty event list fallback included.  
    - Version: 2  

#### 1.5 Data Persistence

- **Overview:** Stores the formatted data in a standardized JSON schema for use by downstream nodes.
- **Nodes Involved:**  
  - `Persist schema`
- **Node Details:**

  - **Persist schema**  
    - Type: Set  
    - Configuration:  
      - Assigns `to`, `eventsList`, and `calendar` fields explicitly from incoming JSON data.  
      - Includes all other fields as well.  
    - Inputs: Receives formatted event list and phone number  
    - Outputs: Passes data to "Basic LLM Chain" node for AI processing  
    - Edge Cases: Requires incoming data to contain all fields; may fail if data missing or malformed.  
    - Retry: Enabled on failure to increase robustness  
    - Version: 3.4  

#### 1.6 AI Summarization

- **Overview:** Uses an AI language model to generate a concise, friendly SMS summary of the day's events based on the formatted event list.
- **Nodes Involved:**  
  - `Basic LLM Chain`
  - (Linked but not triggered in this workflow) `Anthropic Chat Model`
- **Node Details:**

  - **Basic LLM Chain**  
    - Type: Langchain chain LLM node  
    - Configuration:  
      - Input text template includes calendar name and daily events list in XML-like tags.  
      - System message instructs AI persona "Riley" to create a friendly, concise SMS under 200 characters with specific formatting and tone guidelines, including use of emojis and numbered lists.  
      - Retry on failure enabled with 5-second wait between tries.  
    - Inputs: Receives data from "Persist schema" node  
    - Outputs: Passes generated SMS text to "Twilio" node  
    - Edge Cases: AI API quota or authentication errors; prompt formatting errors; possible timeout or output exceeding character limits; fallback behavior not defined here.  
    - Version: 1.6  

  - **Anthropic Chat Model**  
    - Present in workflow but disconnected from main flow; appears configured for Claude 3.7 Sonnet model.  
    - Possibly reserved for alternative AI summarization or testing.  
    - Credentials: Requires Anthropic API key.  
    - No active role in current workflow run.

#### 1.7 SMS Sending

- **Overview:** Sends the SMS summary message to the user’s phone via Twilio.
- **Nodes Involved:**  
  - `Twilio`
- **Node Details:**

  - **Twilio**  
    - Type: Twilio SMS node  
    - Configuration:  
      - `to` number dynamically set from `Persist schema` output  
      - `from` number must be replaced by the user with their purchased Twilio phone number  
      - Message text set dynamically from AI-generated SMS text (`$json.text`)  
      - No additional options enabled  
    - Credentials: Requires configured Twilio API credentials  
    - Inputs: Receives message text and recipient number from "Basic LLM Chain" node  
    - Outputs: None (terminal node)  
    - Edge Cases: Possible failures due to invalid credentials, insufficient Twilio balance, invalid phone numbers, or message length exceeding limits.  
    - Version: 1  

#### 1.8 Documentation and User Guidance (Sticky Notes)

- **Overview:** Provides contextual information, setup instructions, and reminders for the user.
- **Nodes Involved:**  
  - `Sticky Note1` - Explains calendar event fetching and summary purpose  
  - `Sticky Note` (near trigger) - Lists account requirement links (Twilio, Google Cloud)  
  - `Sticky Note2` - Describes AI assistant personality and message formatting  
  - `Sticky Note3` - Reminder to update Twilio sender number  
  - `Sticky Note4` - Workflow goals and personal assistant concept with branding  
- **Role:** Enhance user understanding and ease of configuration; no automated function.

---

### 3. Summary Table

| Node Name                          | Node Type                         | Functional Role                     | Input Node(s)                 | Output Node(s)                          | Sticky Note                                                                                           |
|-----------------------------------|----------------------------------|-----------------------------------|------------------------------|----------------------------------------|-----------------------------------------------------------------------------------------------------|
| Trigger workflow at 7AM            | Schedule Trigger                 | Daily trigger at 7 AM             | None                         | Get events from Events Calendar        | Runs everyday at 7AM                                                                                 |
| Get events from Events Calendar    | Google Calendar                 | Fetch today’s calendar events     | Trigger workflow at 7AM       | Check if there are any events           |                                                                                                     |
| Check if there are any events      | If                              | Check if any events exist         | Get events from Events Calendar| Format list of events and sender mobile num |                                                                                                     |
| Format list of events and sender mobile num | Code (JavaScript)             | Format events into string and set recipient number | Check if there are any events (true branch) | Persist schema                          |                                                                                                     |
| Persist schema                    | Set                             | Store formatted data              | Format list of events and sender mobile num | Basic LLM Chain                        |                                                                                                     |
| Basic LLM Chain                   | Langchain Chain LLM              | Generate friendly SMS summary     | Persist schema               | Twilio                                 | ## Personal Assistant Let's add personality! This is where your events get formatted into a friendly message by the AI. |
| Twilio                           | Twilio SMS Node                  | Send SMS message                 | Basic LLM Chain              | None                                   | ## Send via SMS Don't forget to replace the sender number with the number you just bought from Twilio here! |
| Anthropic Chat Model             | Langchain Chat Model (Claude)   | (Not used actively) alternative AI model | None                      | Basic LLM Chain (ai_languageModel input) |                                                                                                     |
| Sticky Note1                    | Sticky Note                      | Documentation: workflow purpose  | None                         | None                                   | ## Gets events from your Google Calendar Sends daily reminders summarizing the day's events to your phone at 7AM |
| Sticky Note                     | Sticky Note                      | Documentation: account requirements | None                         | None                                   | ## Requirements Create an account for each of these: - [**Twilio account**](https://www.twilio.com/console) (buy a phone number & load it with a few bucks. This will be your assistant's number.) - [**Google Cloud**](https://cloud.google.com/cloud-console?hl=en) (activate the Calendar API so you can get an API key to access your calendar) |
| Sticky Note2                    | Sticky Note                      | Documentation: AI assistant personality | None                         | None                                   | ## Personal Assistant Let's add personality! This is where your events get formatted into a friendly message by the AI. |
| Sticky Note3                    | Sticky Note                      | Documentation: SMS sender number reminder | None                         | None                                   | ## Send via SMS Don't forget to replace the sender number with the number you just bought from Twilio here! |
| Sticky Note4                    | Sticky Note                      | Documentation: workflow goal and branding | None                         | None                                   | ## What does this do? The goal of this automation is for you to receive a daily SMS of the day's calendar events in a summarized form first thing in the morning. This is like a personal assistant giving you a heads up of the day ahead! 🌞 -Anne, [**ralleyreminders.com**](https://www.ralleyreminders.com/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node**
   - Node Type: Schedule Trigger  
   - Configure to trigger daily at 7 AM (set `triggerAtHour` = 7).  
   - Position it as the workflow start node.

2. **Add Google Calendar Node**
   - Node Type: Google Calendar  
   - Operation: getAll  
   - TimeMin: Expression `{{$today}}` (start of current day)  
   - TimeMax: Expression `{{$today.plus(1, 'day')}}` (start of next day)  
   - Select the intended calendar from the list or input calendar ID.  
   - Return All: true  
   - Set Google Calendar OAuth2 credentials properly.

3. **Add If Node to Check Events**
   - Node Type: If  
   - Condition: Check if the number of items from Google Calendar node > 0  
   - Use expression: `{{$items("Get events from Events Calendar").length}} > 0`  
   - True branch leads to event formatting; False branch ends workflow.

4. **Add Code Node to Format Events**
   - Node Type: Code (JavaScript)  
   - Paste the following JS code:
     ```javascript
     const events = $input.all();
     const list = events
       .map(evt => {
         const time = new Date(evt.json.start.dateTime)
           .toLocaleTimeString('en-US', {
             timeZone: evt.json.start.timeZone,
             hour: '2-digit',
             minute: '2-digit'
           });
         return `• ${evt.json.summary} at ${time}`;
       })
       .join("\n");

     return {
       json: {
         to: "+12334567890", // Replace with your recipient's phone number
         eventsList: list || "No events today.",
         calendar: "Anne" // Replace with your calendar or user name
       }
     };
     ```
   - Customize the `to` phone number and `calendar` name.

5. **Add Set Node to Persist Schema**
   - Node Type: Set  
   - Assign variables explicitly:  
     - `to` → `{{$json.to}}`  
     - `eventsList` → `{{$json.eventsList}}`  
     - `calendar` → `{{$json.calendar}}`  
   - Enable "Include All Fields" to pass through other data.

6. **Add Langchain Chain LLM Node (Basic LLM Chain)**
   - Node Type: Langchain Chain LLM  
   - Input Text Template:
     ```
     Today’s calendar items for {{ $json.calendar }}: 
     <daily_events>
     {{ $json.eventsList }}
     </daily_events>

     Now, generate an SMS summary following the guidelines.
     ```
   - System Message (prompt):
     ```
     You are Riley, an AI personal assistant designed to send daily SMS summaries to {{$json.calendar}}. Your task is to create a concise, informative, and friendly message summarizing the day's events for the family.

     The list of the day's events is enclosed in <daily_events />, which may include appointments, activities, reminders, or other important information.  When summarizing the daily events, follow these guidelines:

     1. Prioritize the most important events
     2. Group similar events together
     3. Use concise language to keep the message brief
     4. Include specific times for scheduled events
     5. Mention which family members are involved in each event, if specified.

     Format the SMS in a friendly, conversational tone. Feel free to use emojis. Start with a greeting and end with a positive closing remark, signed in a new line "-Riley from RalleyReminders.com".

     Keep the entire message under 200 characters to ensure it fits in a single SMS. Format it with line breaks. If there is more than one event, number the events to make it easily readable. Here's an example of how your output should be structured:

     <example_sms_one_event>
     Hello {{$json.calendar}}! Today: Dentist appointment @2pm
     Remember to brush your teeth before you go. Have a great day! 😊
     -Riley from RalleyReminders.com
     </example_sms_one_event> 

     <example_sms_many_events>
     Hello {{$json.calendar}}! Today:
     1. Dentist appointment @2pm
     2. Yoga @4pm

     Remember to thank God for this day. Have a great day! 😊
     -Riley from RalleyReminders.com
     </example_sms_many_events>
     ```
   - Enable retry on failure, set wait between tries to 5000 ms.  
   - Configure OpenAI or compatible LLM credentials as per your environment.

7. **Add Twilio Node to Send SMS**
   - Node Type: Twilio  
   - Configure:  
     - `To`: Expression `{{$json.to}}` (from previous node)  
     - `From`: Your Twilio phone number (replace placeholder)  
     - `Message`: Expression `{{$json.text}}` (output of AI node)  
   - Set Twilio API credentials properly.

8. **Connect Nodes in Order**
   - Schedule Trigger → Google Calendar  
   - Google Calendar → If Condition  
   - If Condition (true) → Code Node (Format Events)  
   - Code Node → Set Node (Persist Schema)  
   - Set Node → Basic LLM Chain  
   - Basic LLM Chain → Twilio

9. **Adjust Timezone Settings**
   - Set workflow timezone to user's local timezone (e.g., "America/Edmonton").

10. **Test Workflow**
    - Run manually or wait for scheduled trigger to verify end-to-end SMS sending.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow provides a personal assistant style SMS summary of daily calendar events to enhance daily planning and reminders.                                                                                                  | Workflow purpose summary                                        |
| Twilio account required: purchase a phone number and ensure sufficient balance for SMS sending.                                                                                                                                 | [Twilio Console](https://www.twilio.com/console)               |
| Google Cloud account required: enable Calendar API and create OAuth2 credentials for calendar access.                                                                                                                           | [Google Cloud Console](https://cloud.google.com/cloud-console?hl=en) |
| AI assistant "Riley" is configured to create concise and friendly SMS messages under 200 characters, using specific formatting and tone guidelines.                                                                             | Prompt design best practices                                    |
| Replace all hardcoded phone numbers and calendar names with your own data before deploying.                                                                                                                                     | Important customization note                                    |
| Workflow timezone should be set according to the user's local time for accurate triggering and event retrieval.                                                                                                                | Workflow settings                                               |
| For more information and related tools, visit [RalleyReminders.com](https://www.ralleyreminders.com/)                                                                                                                           | Branding and additional resources                               |

---

**Disclaimer:** This document is derived exclusively from an automated workflow created using n8n, adhering strictly to content policies. It does not contain any illegal, offensive, or protected elements. All data handled is legal and public.