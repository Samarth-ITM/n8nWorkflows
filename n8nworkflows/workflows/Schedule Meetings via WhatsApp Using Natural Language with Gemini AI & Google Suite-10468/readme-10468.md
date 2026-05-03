Schedule Meetings via WhatsApp Using Natural Language with Gemini AI & Google Suite

https://n8nworkflows.xyz/workflows/schedule-meetings-via-whatsapp-using-natural-language-with-gemini-ai---google-suite-10468


# Schedule Meetings via WhatsApp Using Natural Language with Gemini AI & Google Suite

### 1. Workflow Overview

This workflow is designed to automate scheduling meetings through WhatsApp by leveraging natural language understanding powered by Google Gemini AI and Google Suite integrations (Calendar and Sheets). It targets users who want to manage calendar events conversationally via WhatsApp messages, including creating, updating, deleting events, and managing attendees.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures WhatsApp messages to initiate the flow.
- **1.2 Intent Understanding:** Uses AI to interpret user requests and determine scheduling intent.
- **1.3 Contact Lookup:** Retrieves attendee contact details from Google Sheets.
- **1.4 Event Conflict Checking:** Fetches existing calendar events to avoid scheduling conflicts.
- **1.5 Intent Confirmation & Feedback:** Sends interpreted intent back to the user, collects feedback or corrections.
- **1.6 Intent Correction:** Adjusts intent based on user feedback using AI.
- **1.7 Calendar Operations:** Executes create, update, or delete operations on Google Calendar.
- **1.8 Notification & Response:** Sends WhatsApp confirmations and notifies attendees.
- **1.9 Auxiliary:** Includes helper nodes like code execution and the central AI chat model powering all agents.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives incoming WhatsApp messages to trigger the workflow.
- **Nodes Involved:**  
  - WhatsApp Trigger
  - Sticky Note (Capture Request)

- **Node Details:**

  - **WhatsApp Trigger**  
    - Type: Trigger node for WhatsApp Cloud API messages  
    - Configuration: Listens for message updates on WhatsApp; webhook ID assigned  
    - Inputs: Incoming WhatsApp message JSON  
    - Outputs: Passes message JSON downstream  
    - Edge Cases: Webhook misconfiguration, WhatsApp API downtime, message format changes

  - **Sticky Note (Capture Request)**  
    - Type: Visual annotation for clarity  
    - Content: "# Capture Request"

---

#### 2.2 Intent Understanding

- **Overview:** Uses Google Gemini-based AI agent to analyze the user's natural language message and extract the scheduling intent.
- **Nodes Involved:**  
  - Intent Agent  
  - Google Gemini Chat Model  
  - Contacts Data  
  - Get_Events  
  - Set Intent  
  - Sticky Note (Chat Model)  

- **Node Details:**

  - **Intent Agent**  
    - Type: AI Agent node (LangChain) powered by Google Gemini  
    - Configuration: Uses the user's WhatsApp message text as input  
    - System Message: Detailed instructions guiding the AI on interpreting calendar-related intents, including use of tools (Contacts Data, Get_Events) and rules for conflict checking  
    - Input: User message text from WhatsApp Trigger  
    - Output: Structured intent including event details and attendee emails  
    - Edge Cases: Ambiguous user messages, missing attendee info, conflicting event times

  - **Google Gemini Chat Model**  
    - Type: Language model node for AI processing  
    - Role: Provides the NLP capabilities for all AI agents in the workflow  
    - Configuration: Requires Google Gemini API key and proper credentials  
    - Connected to Intent Agent, Check Feedback, Correction Agent, Calendar Agent

  - **Contacts Data**  
    - Type: Google Sheets node  
    - Configuration: Reads a specified Google Sheet (spreadsheet ID and sheet name) containing contact information (Name, Email, Type)  
    - Role: Supplies contact emails for attendees found in user requests  
    - Input: Invoked as a tool by Intent Agent  
    - Edge Cases: Missing or outdated contacts, multiple contacts with same name

  - **Get_Events**  
    - Type: Google Calendar Tool node  
    - Configuration: Retrieves all events on the day requested by the user to check for conflicts  
    - Inputs: Time window derived from user request (start and end of the requested day)  
    - Output: List of events to inform conflict resolution  
    - Edge Cases: API rate limits, calendar access permissions, incorrect date parsing

  - **Set Intent**  
    - Type: Set node  
    - Configuration: Saves the interpreted intent from Intent Agent output into the workflow context as `intent` for downstream use  
    - Input: Output of Intent Agent  
    - Output: Passes intent for confirmation message generation

  - **Sticky Note (Chat Model)**  
    - Type: Visual annotation  
    - Content: "## Chat Model"

---

#### 2.3 Contact Lookup

- **Overview:** Fetches user contact details from a Google Sheets document referenced by the AI agents for attendee email resolution.
- **Nodes Involved:**  
  - Contacts Data (shared with Intent Agent as a tool)

- **Node Details:**  
  (See description above in 2.2)

---

#### 2.4 Event Conflict Checking

- **Overview:** Checks user's Google Calendar for existing events at the requested time to avoid double bookings.
- **Nodes Involved:**  
  - Get_Events (shared with Intent Agent as a tool)

- **Node Details:**  
  (See description above in 2.2)

---

#### 2.5 Intent Confirmation & Feedback

- **Overview:** Sends the interpreted intent back to the user via WhatsApp and waits for feedback or approval to proceed.
- **Nodes Involved:**  
  - Send message and wait for response  
  - Check Feedback  
  - Sticky Note (Validate Details)

- **Node Details:**

  - **Send message and wait for response**  
    - Type: WhatsApp node (sendAndWait operation)  
    - Configuration: Sends the AI-generated intent explanation to the user, waits for free text response  
    - Inputs: Message text from Set Intent `intent` field, recipient phone number from WhatsApp Trigger  
    - Output: User feedback text  
    - Edge Cases: WhatsApp API errors, user no response, message formatting issues

  - **Check Feedback**  
    - Type: Text Classifier (LangChain)  
    - Configuration: Classifies user feedback into "Approved" or "Denied" categories with example phrases  
    - Input: User feedback text  
    - Output: Classification result to decide next steps (proceed or correct)  
    - Edge Cases: Ambiguous feedback, unexpected user replies

  - **Sticky Note (Validate Details)**  
    - Visual annotation with content: "# Validate Details"

---

#### 2.6 Intent Correction

- **Overview:** If the user denies or requests changes, this block refines the intent using AI based on user's correction.
- **Nodes Involved:**  
  - Correction Agent  
  - Set Intent

- **Node Details:**

  - **Correction Agent**  
    - Type: AI Agent node (LangChain) powered by Google Gemini  
    - Configuration: Takes original intent and user feedback to output corrected intent with full details, or ask clarifying questions if needed  
    - Inputs: Original intent from Set Intent, user feedback from Check Feedback  
    - Output: Updated intent for further processing  
    - Edge Cases: Conflicting feedback, incomplete correction details

  - **Set Intent** (second usage)  
    - Stores the corrected intent for further use in calendar operations

---

#### 2.7 Calendar Operations

- **Overview:** Performs actual calendar actions (create, update, delete) on Google Calendar based on final intent.
- **Nodes Involved:**  
  - Calendar Agent  
  - Create Event  
  - Create Event with Attendee  
  - Update Event  
  - Delete Event  
  - Code (helper)  
  - Sticky Note (Schedule Event & Notify)

- **Node Details:**

  - **Calendar Agent**  
    - Type: AI Agent node (LangChain) powered by Google Gemini  
    - Configuration: Converts intent into API calls for Calendar operations following rules: use attendee-specific or solo event creation tools, always get event info before update/delete  
    - Input: Final intent JSON  
    - Output: Decision and data to drive Google Calendar nodes  
    - Edge Cases: API failures, missing event IDs, concurrency issues

  - **Create Event**  
    - Type: Google Calendar Tool node  
    - Operation: Create new event without attendees  
    - Inputs: Start, end times, summary derived from AI fields  
    - Output: Confirmation of event creation

  - **Create Event with Attendee**  
    - Type: Google Calendar Tool node  
    - Operation: Create event including attendee emails  
    - Inputs: Start, end times, summary, attendees list  
    - Output: Confirmation and attendee invitations

  - **Update Event**  
    - Type: Google Calendar Tool node  
    - Operation: Update existing event by eventId  
    - Inputs: eventId, new start/end times, other update fields  
    - Output: Confirmation of update

  - **Delete Event**  
    - Type: Google Calendar Tool node  
    - Operation: Delete event by eventId  
    - Inputs: eventId  
    - Output: Confirmation of deletion

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Adds a field `myNewField` with value 1 to all input items; likely a placeholder or debugging aid  
    - Input/Output: Passes data downstream unaltered except for new field addition

  - **Sticky Note (Schedule Event & Notify)**  
    - Visual annotation: "# Schedule Event & Notify"

---

#### 2.8 Notification & Response

- **Overview:** Sends final confirmation messages back to the user on WhatsApp after calendar operations complete.
- **Nodes Involved:**  
  - Response (WhatsApp node)

- **Node Details:**

  - **Response**  
    - Type: WhatsApp node (send operation)  
    - Configuration: Sends the output message text (confirmation or updates) to the user who initiated the conversation  
    - Inputs: Text from Code node or Calendar Agent output  
    - Edge Cases: WhatsApp API delivery failures, incorrect phone numbers

---

#### 2.9 Auxiliary & Metadata

- **Nodes Involved:**  
  - Sticky Notes (branding, instructions, setup guides)  
  - Sticky Note4: Project credit  
  - Sticky Note7: Comprehensive workflow description  
  - Sticky Note5: Setup instructions

- These provide valuable documentation, instructions, and credits embedded as visual notes but do not affect execution.

---

### 3. Summary Table

| Node Name                    | Node Type                              | Functional Role                          | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                     |
|------------------------------|--------------------------------------|----------------------------------------|------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| WhatsApp Trigger             | WhatsApp Trigger                     | Captures incoming WhatsApp messages    |                              | Intent Agent                      | # Capture Request                                                                             |
| Intent Agent                | LangChain AI Agent                   | Understand user intent from message    | WhatsApp Trigger, Contacts Data, Get_Events, Google Gemini Chat Model | Set Intent                        |                                                                                                |
| Contacts Data               | Google Sheets Tool                   | Provides attendee contact details      |                              | Intent Agent                     |                                                                                                |
| Get_Events                  | Google Calendar Tool                 | Checks calendar for existing events    |                              | Intent Agent                     |                                                                                                |
| Set Intent                  | Set Node                           | Stores interpreted intent               | Intent Agent, Correction Agent | Send message and wait for response |                                                                                                |
| Send message and wait for response | WhatsApp                        | Sends intent for user confirmation     | Set Intent                   | Check Feedback                  | # Validate Details                                                                           |
| Check Feedback              | LangChain Text Classifier            | Classifies user feedback (approve/deny) | Send message and wait for response | Calendar Agent, Correction Agent |                                                                                                |
| Correction Agent            | LangChain AI Agent                   | Adjusts intent based on feedback       | Check Feedback               | Set Intent                      |                                                                                                |
| Calendar Agent              | LangChain AI Agent                   | Executes calendar actions               | Set Intent, Check Feedback   | Code                           | # Schedule Event & Notify                                                                    |
| Create Event                | Google Calendar Tool                 | Creates solo calendar event             | Calendar Agent               | Calendar Agent                  |                                                                                                |
| Create Event with Attendee  | Google Calendar Tool                 | Creates event with attendees            | Calendar Agent               | Calendar Agent                  |                                                                                                |
| Update Event                | Google Calendar Tool                 | Updates existing calendar event         | Calendar Agent               | Calendar Agent                  |                                                                                                |
| Delete Event                | Google Calendar Tool                 | Deletes calendar event                   | Calendar Agent               | Calendar Agent                  |                                                                                                |
| Code                       | Code Node (JavaScript)               | Helper node, adds field to data         | Calendar Agent               | Response                       |                                                                                                |
| Response                   | WhatsApp                            | Sends final confirmation message        | Code                        |                               |                                                                                                |
| Google Gemini Chat Model    | LangChain Language Model             | Provides AI NLP capabilities            |                              | Intent Agent, Check Feedback, Correction Agent, Calendar Agent | ## Chat Model                                                                              |
| Sticky Note                | Sticky Note                         | Visual annotations                       |                              |                               | Multiple notes: # Capture Request, # Validate Details, # Schedule Event & Notify, branding & setup instructions |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure: Connect to WhatsApp Cloud API, set webhook, listen for messages  
   - Output: Incoming WhatsApp message JSON

2. **Create Contacts Data Node**  
   - Type: Google Sheets Tool  
   - Configure: Connect to your Google Sheets account  
   - Document ID: Set to your contacts spreadsheet ID  
   - Sheet Name: Set to your contacts sheet tab (e.g., `gid=0`)  
   - Output: Contact information for attendee lookup

3. **Create Get_Events Node**  
   - Type: Google Calendar Tool  
   - Operation: Get All Events  
   - Calendar: Set your Google calendar ID or email  
   - TimeMin & TimeMax: Set dynamically based on user requested date (use expressions)  
   - Output: List of events on requested day

4. **Create Google Gemini Chat Model Node**  
   - Type: LangChain Google Gemini Chat Model  
   - Configure your Google Gemini API credentials  
   - No inputs, but will be linked to all AI agents

5. **Create Intent Agent Node**  
   - Type: LangChain Agent  
   - Input: User message text from WhatsApp Trigger (expression: `{{ $json.messages[0].text.body }}`)  
   - Tools: Add Contacts Data and Get_Events nodes as tools accessible by the agent  
   - System Message: Provide instructions for calendar intent understanding including tools usage and rules  
   - Link AI Language Model: Google Gemini Chat Model  
   - Output: Structured intent JSON

6. **Create Set Intent Node**  
   - Type: Set  
   - Assign `intent` field from Intent Agent output (expression: `{{$json.output}}`)  
   - Output: Intent for confirmation message

7. **Create Send message and wait for response Node**  
   - Type: WhatsApp node (sendAndWait operation)  
   - Message: Use intent text from Set Intent node  
   - Phone Number ID & Recipient: Use from WhatsApp Trigger incoming message metadata and sender  
   - Wait for free text response

8. **Create Check Feedback Node**  
   - Type: LangChain Text Classifier  
   - Input Text: User feedback text from WhatsApp response  
   - Categories: "Approved" and "Denied" with example phrases  
   - Output: Feedback classification

9. **Create Correction Agent Node**  
   - Type: LangChain Agent  
   - Input: Original intent from Set Intent and user feedback text  
   - System Message: Instructions to correct or confirm intent, request clarification if needed  
   - Link AI Language Model: Google Gemini Chat Model  
   - Output: Corrected intent

10. **Create second Set Intent Node**  
    - Stores corrected intent from Correction Agent output

11. **Create Calendar Agent Node**  
    - Type: LangChain Agent  
    - Input: Final intent JSON from Set Intent  
    - System Message: Instructions to perform calendar actions (create, update, delete) using Google Calendar nodes  
    - Tools: Connect Create Event, Create Event with Attendee, Update Event, Delete Event nodes as tools  
    - Link AI Language Model: Google Gemini Chat Model

12. **Create Google Calendar Nodes**  
    - Create Event: Create solo event with start, end, summary from AI intent  
    - Create Event with Attendee: Same as Create Event but includes attendees list  
    - Update Event: Updates event by eventId with new details  
    - Delete Event: Deletes event by eventId  
    - All Calendar Nodes: Connect to your Google Calendar credentials and calendar ID

13. **Create Code Node**  
    - JavaScript to add a field or perform minor data manipulation (optional)  
    - Connect output of Calendar Agent to this node

14. **Create Response Node**  
    - WhatsApp node (send operation)  
    - Sends final confirmation message from Code node or Calendar Agent output to user

15. **Connect nodes in order:**  
    WhatsApp Trigger → Intent Agent → Set Intent → Send message and wait for response → Check Feedback → (If Denied) Correction Agent → Set Intent → Calendar Agent → Calendar operation nodes → Code → Response

16. **Add Sticky Notes** for documentation and visual clarity, including:  
    - # Capture Request near WhatsApp Trigger  
    - # Validate Details near Send message and wait for response  
    - ## Chat Model near Google Gemini Chat Model  
    - # Schedule Event & Notify near Calendar Agent and Google Calendar nodes  
    - Branding and setup instructions as needed

17. **Credentials Setup:**  
    - WhatsApp Cloud API credentials in WhatsApp nodes  
    - Google OAuth2 credentials for Google Calendar and Google Sheets nodes  
    - Google Gemini API key for AI model node and linked agents

18. **Test the workflow:**  
    - Send WhatsApp messages like “schedule meeting with John at 10am on 27 August”  
    - Verify AI intent recognition, contact lookup, conflict checking, event creation, and WhatsApp confirmation

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow enables fully automated meeting scheduling via WhatsApp using Google Gemini AI and Google Suite integrations.                                      | Workflow description and use case                                                              |
| Setup instructions included as sticky note (# ⚙️ Setup Instructions), detailing required integrations and step-by-step configuration.                      | See Sticky Note5 content                                                                        |
| AI prompt instructions guide agents on intent understanding, correction, and calendar operation rules, ensuring natural language processing accuracy.       | Embedded in system messages of AI Agent nodes                                                  |
| Contact management is handled through a Google Sheet acting as a contact database with columns for Name, Email, and Type.                                    | Contacts Data node configuration                                                               |
| The workflow avoids double bookings by querying Google Calendar events on the requested date before scheduling new events.                                  | Get_Events node purpose                                                                         |
| Confirmation and correction loop with user via WhatsApp enables iterative refinement of scheduling requests.                                                | Send message and wait for response, Check Feedback, Correction Agent nodes                      |
| Project credit: Muhammad Ali Zubair | AI Automation Expert                                                                                            | Sticky Note4 content                                                                           |
| Video and detailed project explanation available in embedded sticky notes.                                                                                  | Sticky Note7 content                                                                           |
| Optional enhancements suggested include Slack notifications, timezone handling, and recurring meeting support.                                             | Sticky Note5 setup instructions                                                                |

---

This detailed breakdown enables users and automation agents to fully understand, reproduce, and adapt the workflow efficiently while anticipating common points of failure and integration nuances.