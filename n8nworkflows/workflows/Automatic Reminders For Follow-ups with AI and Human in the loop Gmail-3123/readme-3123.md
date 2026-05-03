Automatic Reminders For Follow-ups with AI and Human in the loop Gmail

https://n8nworkflows.xyz/workflows/automatic-reminders-for-follow-ups-with-ai-and-human-in-the-loop-gmail-3123


# Automatic Reminders For Follow-ups with AI and Human in the loop Gmail

### 1. Workflow Overview

This workflow automates follow-up reminders for sales meetings by combining AI-driven scheduling suggestions with human approval before booking. It targets sales professionals or teams who want to ensure timely re-engagement with prospects after meetings, reducing manual task load while maintaining control over outreach.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Event Retrieval:** Periodically fetch past sales meetings from Google Calendar within a specific date range.
- **1.2 Follow-up Detection:** For each meeting, check Gmail for any follow-up emails exchanged with the lead since the meeting ended.
- **1.3 AI Availability Suggestion:** For meetings lacking follow-up, use an AI agent to analyze the previous meeting and suggest similar available slots for the next call.
- **1.4 Human-in-the-Loop Approval:** Send the suggested slots and a reminder email to the user via Gmail’s send-and-wait-for-approval mode, allowing the user to accept, suggest alternatives, or decline.
- **1.5 AI Meeting Booking:** If approved, another AI agent books the meeting in Google Calendar; if declined, no action is taken.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Event Retrieval

**Overview:**  
This block triggers the workflow daily and retrieves past sales meetings from Google Calendar that occurred 2 to 4 days ago. It also removes duplicates to avoid reprocessing the same events.

**Nodes Involved:**  
- Schedule Trigger  
- Get Past Events  
- Mark as Seen  
- Loop Over Items

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow daily at 6 AM.  
  - Configuration: Fires once daily at hour 6.  
  - Inputs: None  
  - Outputs: Triggers "Get Past Events"  
  - Edge Cases: Timezone misconfiguration could cause missed events.

- **Get Past Events**  
  - Type: Google Calendar  
  - Role: Fetches all calendar events between 2 and 4 days ago.  
  - Configuration:  
    - timeMin: 4 days ago  
    - timeMax: 2 days ago  
    - Calendar ID: User must replace `<your-calendar>` with their calendar ID.  
    - Operation: getAll events  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Events list to "Mark as Seen"  
  - Edge Cases: Calendar permission errors, empty event sets.

- **Mark as Seen**  
  - Type: Remove Duplicates  
  - Role: Prevents reprocessing of events already handled in previous runs by deduplicating on event ID.  
  - Configuration: Deduplication based on event `id`.  
  - Inputs: Events from "Get Past Events"  
  - Outputs: Unique events to "Loop Over Items"  
  - Edge Cases: If event IDs change or are missing, duplicates may occur.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each event individually downstream.  
  - Configuration: Default batch size (processes items one by one).  
  - Inputs: Unique events from "Mark as Seen"  
  - Outputs: Individual event to "Only Follow Ups" and "Get Emails Since"  
  - Edge Cases: Large event sets may slow processing.

---

#### 1.2 Follow-up Detection

**Overview:**  
For each past meeting, this block checks Gmail for any emails exchanged with the lead since the meeting ended. If no emails are found, the meeting is flagged for follow-up.

**Nodes Involved:**  
- Only Follow Ups  
- Get Emails Since  
- Flag to Follow Up

**Node Details:**

- **Only Follow Ups**  
  - Type: Filter  
  - Role: Filters events to those requiring follow-up (initially configured with an empty condition, effectively passing all items; likely a placeholder or to be customized).  
  - Inputs: Events from "Loop Over Items"  
  - Outputs: Events to "Meeting Availability Agent" (for follow-up)  
  - Edge Cases: Misconfiguration may block all or no events.

- **Get Emails Since**  
  - Type: Gmail  
  - Role: Searches Gmail threads involving the lead’s email (attendee other than self) sent or received after the meeting end time.  
  - Configuration:  
    - Query: from or to the lead’s email  
    - ReceivedAfter: meeting end date/time  
    - Limit: 1 (only checks if any email exists)  
    - Resource: thread  
  - Inputs: Events from "Loop Over Items"  
  - Outputs: Email threads to "Flag to Follow Up"  
  - Edge Cases: Gmail API rate limits, auth errors, incorrect email extraction.

- **Flag to Follow Up**  
  - Type: Set  
  - Role: Adds a `followUp` boolean flag to the event JSON, true if no emails found (empty Gmail result).  
  - Configuration: Uses expression to set `followUp` based on whether Gmail search result is empty.  
  - Inputs: Email search results from "Get Emails Since"  
  - Outputs: Events with followUp flag to "Loop Over Items" (to reprocess with updated flag)  
  - Edge Cases: Expression failures if input data structure changes.

---

#### 1.3 AI Availability Suggestion

**Overview:**  
For meetings flagged for follow-up, this block uses an AI agent to analyze the previous meeting details and suggest available calendar slots for the next meeting, aiming for similar timing and duration.

**Nodes Involved:**  
- Meeting Availability Agent  
- Model  
- Availability  
- Output  
- Generate Message

**Node Details:**

- **Meeting Availability Agent**  
  - Type: LangChain AI Agent  
  - Role: Analyzes previous meeting details and suggests available slots for the next meeting.  
  - Configuration:  
    - Input text includes meeting title, date, timezone, and duration.  
    - System message instructs to find similar future slots considering day, time, and duration.  
    - Output parser enabled with a JSON schema expecting an array of slots with start and end times.  
  - Inputs: Events from "Only Follow Ups"  
  - Outputs: Suggested slots to "Generate Message"  
  - Edge Cases: AI model errors, invalid date/time parsing, empty slot suggestions.

- **Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides the language model backend for the AI agent.  
  - Configuration: Uses GPT-4o-mini model.  
  - Inputs: From "Meeting Availability Agent" (ai_languageModel)  
  - Outputs: To "Meeting Availability Agent" (ai_languageModel)  
  - Credentials: OpenAI API key required.  
  - Edge Cases: API rate limits, auth failures.

- **Availability**  
  - Type: Google Calendar Tool  
  - Role: Checks calendar availability for suggested time ranges.  
  - Configuration:  
    - Timezone, timeMin, timeMax dynamically set from AI agent overrides.  
    - Calendar ID must be set by user.  
  - Inputs: From "Meeting Availability Agent" (ai_tool)  
  - Outputs: Availability info to "Meeting Availability Agent" (ai_tool)  
  - Credentials: Google Calendar OAuth2 required.  
  - Edge Cases: Calendar permission errors, incorrect timezone handling.

- **Output**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI agent’s raw output into structured JSON according to defined schema.  
  - Configuration: JSON schema expects an array of slots with start and end strings.  
  - Inputs: From "Meeting Availability Agent" (ai_outputParser)  
  - Outputs: Parsed slots to "Generate Message"  
  - Edge Cases: Parsing errors if AI output is malformed.

- **Generate Message**  
  - Type: Set  
  - Role: Constructs a human-readable message listing available slots for follow-up, excluding weekends.  
  - Configuration:  
    - Uses expressions to format slot dates/times and insert lead email.  
  - Inputs: Parsed slots from "Output" and event data from "Only Follow Ups"  
  - Outputs: Message text to "Send for Human Approval"  
  - Edge Cases: Date formatting errors, empty slot list.

---

#### 1.4 Human-in-the-Loop Approval

**Overview:**  
This block sends the AI-generated follow-up message and available slots to the user via Gmail’s send-and-wait-for-approval mode. The user replies with natural language to accept, suggest alternatives, or decline.

**Nodes Involved:**  
- Send for Human Approval

**Node Details:**

- **Send for Human Approval**  
  - Type: Gmail  
  - Role: Sends the follow-up message to the user and waits for a free-text response.  
  - Configuration:  
    - SendTo: User’s email (must be replaced by user)  
    - Subject: Includes lead email for context  
    - Message: From "Generate Message" node  
    - Operation: sendAndWait (human-in-the-loop)  
    - ResponseType: freeText (user can reply with any text)  
  - Inputs: Message from "Generate Message"  
  - Outputs: User response to "Meeting Booking Agent"  
  - Credentials: Gmail OAuth2 required.  
  - Edge Cases: Email delivery failures, user non-response, ambiguous replies.

---

#### 1.5 AI Meeting Booking

**Overview:**  
Based on the user’s reply, this AI agent either books the meeting in Google Calendar or takes no action if declined.

**Nodes Involved:**  
- Meeting Booking Agent  
- Model1  
- Meetings

**Node Details:**

- **Meeting Booking Agent**  
  - Type: LangChain AI Agent  
  - Role: Interprets user response and books the meeting if approved.  
  - Configuration:  
    - Input text: User’s free-text reply from Gmail node.  
    - System message instructs to book the meeting if confirmed or do nothing if declined.  
    - Includes the original AI-generated message for context.  
  - Inputs: User response from "Send for Human Approval"  
  - Outputs: Booking instructions to "Meetings"  
  - Edge Cases: Misinterpretation of user intent, booking conflicts.

- **Model1**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides language model backend for booking agent.  
  - Configuration: Uses GPT-4o-mini model.  
  - Inputs: From "Meeting Booking Agent" (ai_languageModel)  
  - Outputs: To "Meeting Booking Agent" (ai_languageModel)  
  - Credentials: OpenAI API key required.  
  - Edge Cases: API limits, auth errors.

- **Meetings**  
  - Type: Google Calendar Tool  
  - Role: Creates the calendar event for the follow-up meeting.  
  - Configuration:  
    - Start, End, Summary, Description dynamically set from AI agent overrides.  
    - Calendar ID must be set by user.  
  - Inputs: Booking instructions from "Meeting Booking Agent" (ai_tool)  
  - Outputs: Final booking confirmation (not used downstream)  
  - Credentials: Google Calendar OAuth2 required.  
  - Edge Cases: Calendar conflicts, permission errors.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                             | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                                                  |
|-------------------------|----------------------------------|---------------------------------------------|-------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                 | Triggers workflow daily at 6 AM              | None                    | Get Past Events                | ## 1. Get Recent Meetings<br>[Learn more about the GCalendar node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar)<br>Scheduled trigger pulls past meetings 2-3 days ago. |
| Get Past Events         | Google Calendar                  | Retrieves past meetings from calendar        | Schedule Trigger         | Mark as Seen                  | See above                                                                                                                    |
| Mark as Seen            | Remove Duplicates                | Removes duplicate events to avoid reprocessing | Get Past Events          | Loop Over Items               | See above                                                                                                                    |
| Loop Over Items         | Split In Batches                 | Processes each event individually             | Mark as Seen             | Only Follow Ups, Get Emails Since | See above                                                                                                                    |
| Only Follow Ups         | Filter                          | Filters events needing follow-up              | Loop Over Items          | Meeting Availability Agent    | ## 2. Check If Any Messages Since<br>[Read more about the Gmail node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail)<br>Checks if any emails exchanged since meeting. |
| Get Emails Since        | Gmail                           | Searches Gmail for emails after meeting       | Loop Over Items          | Flag to Follow Up             | See above                                                                                                                    |
| Flag to Follow Up       | Set                             | Flags events for follow-up if no emails found | Get Emails Since         | Loop Over Items               | See above                                                                                                                    |
| Meeting Availability Agent | LangChain AI Agent             | Suggests available slots for next meeting     | Only Follow Ups          | Generate Message              | ## 3. Suggest Availability For Next Call<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/)<br>AI suggests similar meeting slots. |
| Model                   | LangChain OpenAI Chat Model     | Provides AI model for availability agent     | Meeting Availability Agent | Meeting Availability Agent    | See above                                                                                                                    |
| Availability            | Google Calendar Tool            | Checks calendar availability                   | Meeting Availability Agent | Meeting Availability Agent    | See above                                                                                                                    |
| Output                  | LangChain Output Parser Structured | Parses AI output into structured JSON         | Meeting Availability Agent | Generate Message              | See above                                                                                                                    |
| Generate Message        | Set                             | Creates follow-up message with available slots | Output, Only Follow Ups  | Send for Human Approval       | ## 4. Get Human Approval<br>[Learn more about n8n's Human-in-the-loop features](https://docs.n8n.io/advanced-ai/examples/human-fallback/)<br>Uses Gmail send-and-wait-for-approval mode. |
| Send for Human Approval | Gmail                           | Sends message to user and waits for reply     | Generate Message         | Meeting Booking Agent         | See above                                                                                                                    |
| Meeting Booking Agent   | LangChain AI Agent              | Books meeting if user approves                 | Send for Human Approval  | Meetings                     | ## 5. Book the meeting If Accepted<br>[Learn more about the AI Agent node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/)<br>Books meeting or does nothing if declined. |
| Model1                  | LangChain OpenAI Chat Model     | Provides AI model for booking agent            | Meeting Booking Agent    | Meeting Booking Agent         | See above                                                                                                                    |
| Meetings                | Google Calendar Tool            | Creates calendar event for follow-up meeting  | Meeting Booking Agent    | None                        | See above                                                                                                                    |
| Sticky Note             | Sticky Note                    | Documentation note                             | None                    | None                        | See section 5                                                                                                               |
| Sticky Note1            | Sticky Note                    | Documentation note                             | None                    | None                        | See section 5                                                                                                               |
| Sticky Note2            | Sticky Note                    | Documentation note                             | None                    | None                        | See section 5                                                                                                               |
| Sticky Note3            | Sticky Note                    | Documentation note                             | None                    | None                        | See section 5                                                                                                               |
| Sticky Note4            | Sticky Note                    | Documentation note                             | None                    | None                        | See section 5                                                                                                               |
| Sticky Note5            | Sticky Note                    | Documentation note                             | None                    | None                        | See section 5                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 6:00 AM.

2. **Create Google Calendar node "Get Past Events"**  
   - Operation: getAll events  
   - timeMin: `={{ $now.minus({ day: 4 }) }}`  
   - timeMax: `={{ $now.minus({ day: 2 }) }}`  
   - Calendar: Set your calendar ID (replace `<your-calendar>`)  
   - Connect input from Schedule Trigger.

3. **Create Remove Duplicates node "Mark as Seen"**  
   - Operation: removeItemsSeenInPreviousExecutions  
   - Deduplication value: `={{ $json.id }}`  
   - Connect input from Get Past Events.

4. **Create Split In Batches node "Loop Over Items"**  
   - Default batch size (1)  
   - Connect input from Mark as Seen.

5. **Create Filter node "Only Follow Ups"**  
   - Configure filter to pass only events flagged for follow-up (initially empty, to be customized).  
   - Connect input from Loop Over Items.

6. **Create Gmail node "Get Emails Since"**  
   - Operation: search threads  
   - Limit: 1  
   - Filters:  
     - Query: `=(from:{{ $json.attendees.find(attendee => !attendee.self)?.email }} OR to:{{ $json.attendees.find(attendee => !attendee.self)?.email }})`  
     - ReceivedAfter: `={{ $json.end.dateTime }}`  
   - Connect input from Loop Over Items.

7. **Create Set node "Flag to Follow Up"**  
   - Add field `followUp` with expression:  
     ```js
     {
       ...$('Loop Over Items').first().json,
       followUp: $json.isEmpty()
     }
     ```  
   - Connect input from Get Emails Since.

8. **Connect output of Flag to Follow Up back to Loop Over Items**  
   - This allows reprocessing with updated follow-up flag.

9. **Connect output of Only Follow Ups to "Meeting Availability Agent" node**  
   - Create LangChain AI Agent node "Meeting Availability Agent"  
   - Input text includes meeting details (title, date, timezone, duration).  
   - System message instructs to find similar future slots.  
   - Enable output parser with JSON schema for slots.  
   - Connect AI language model node "Model" (LangChain OpenAI Chat) with GPT-4o-mini model to this agent.  
   - Connect Google Calendar Tool node "Availability" to check calendar availability for suggested slots.  
   - Connect Output Parser node "Output" to parse AI agent output.

10. **Create Set node "Generate Message"**  
    - Compose message listing available slots (exclude weekends), including lead email.  
    - Connect input from Output and Only Follow Ups.

11. **Create Gmail node "Send for Human Approval"**  
    - Operation: sendAndWait  
    - SendTo: your email address (replace `<your-email-here>`)  
    - Subject: includes lead email  
    - Message: from Generate Message  
    - ResponseType: freeText  
    - Connect input from Generate Message.

12. **Create LangChain AI Agent node "Meeting Booking Agent"**  
    - Input text: user reply from Gmail node  
    - System message instructs to book meeting if confirmed, else do nothing.  
    - Connect AI language model node "Model1" (LangChain OpenAI Chat) with GPT-4o-mini model.  
    - Connect Google Calendar Tool node "Meetings" to create calendar event with start, end, summary, description from AI agent overrides.  
    - Connect input from Send for Human Approval.

13. **Set credentials**  
    - Google Calendar nodes: configure with Google OAuth2 credentials with calendar access.  
    - Gmail nodes: configure with Gmail OAuth2 credentials.  
    - LangChain OpenAI nodes: configure with OpenAI API key.

14. **Replace all placeholder calendar IDs and email addresses with your actual values.**

15. **Test the workflow by running manually or waiting for scheduled trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow extends sales follow-up reminders by combining AI suggestions with human approval to reduce manual tasks while maintaining control.                                                                                                                                                                                                          | Workflow purpose                                                                                         |
| Scheduled trigger runs daily at 6 AM to fetch meetings from 2-4 days ago.                                                                                                                                                                                                                                                                                     | Section 1.1                                                                                             |
| Gmail node uses send-and-wait-for-approval mode to enable human-in-the-loop interaction.                                                                                                                                                                                                                                                                    | Section 1.4                                                                                             |
| AI Agents use GPT-4o-mini model to suggest meeting slots and book meetings based on user input.                                                                                                                                                                                                                                                             | Sections 1.3 and 1.5                                                                                   |
| Update calendar nodes and Gmail nodes with correct calendar IDs and email addresses before use.                                                                                                                                                                                                                                                             | Usage instructions                                                                                      |
| Consider swapping Google Calendar and Gmail nodes with Microsoft Outlook equivalents if needed.                                                                                                                                                                                                                                                             | Customization note                                                                                      |
| For alternative human approval channels, consider integrating Telegram or WhatsApp nodes with send-and-wait features.                                                                                                                                                                                                                                       | Customization note                                                                                      |
| Join the n8n community for support: [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)                                                                                                                                                                                                                                      | Support links                                                                                           |
| Google Calendar node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar                                                                                                                                                                                                                                        | Sticky Note 1                                                                                           |
| Gmail node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail                                                                                                                                                                                                                                                           | Sticky Note 2                                                                                           |
| AI Agent node documentation: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/                                                                                                                                                                                                                                    | Sticky Notes 3 and 5                                                                                     |
| Human-in-the-loop feature documentation: https://docs.n8n.io/advanced-ai/examples/human-fallback/                                                                                                                                                                                                                                                            | Sticky Note 4                                                                                           |

---

This detailed reference document enables advanced users and AI agents to fully understand, reproduce, and customize the "Automatic Reminders For Follow-ups with AI and Human in the loop Gmail" workflow.