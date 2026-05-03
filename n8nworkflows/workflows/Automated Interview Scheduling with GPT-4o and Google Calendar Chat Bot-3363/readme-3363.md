Automated Interview Scheduling with GPT-4o and Google Calendar Chat Bot

https://n8nworkflows.xyz/workflows/automated-interview-scheduling-with-gpt-4o-and-google-calendar-chat-bot-3363


# Automated Interview Scheduling with GPT-4o and Google Calendar Chat Bot

### 1. Workflow Overview

This workflow, titled **Automated Interview Scheduling with GPT-4o and Google Calendar Chat Bot**, enables candidates to schedule interviews via a conversational AI assistant integrated with Google Calendar. It automates the process of checking calendar availability, proposing available 30-minute weekday time slots between 9 AM and 5 PM Eastern Time, and booking confirmed interview meetings.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Conversational AI Interaction**  
  Handles incoming chat messages from candidates, maintains conversational context, and guides users through providing interview details.

- **1.2 AI Processing and Scheduling Logic**  
  Uses GPT-4o to interpret user inputs, validate interview details, and coordinate availability checks.

- **1.3 Calendar Availability Retrieval and Slot Generation**  
  Reads existing Google Calendar events, splits them into 30-minute blocks, generates potential interview slots, and merges these to identify free times.

- **1.4 Date Interpretation Tool**  
  Supports natural language date expressions (e.g., "next Tuesday") by providing a list of upcoming weekdays.

- **1.5 Booking Confirmation and Calendar Event Creation**  
  Converts AI output into structured JSON, creates the calendar event, and sends a confirmation message back to the candidate.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Conversational AI Interaction

**Overview:**  
This block receives chat messages from candidates via a public webhook, maintains session memory for context, and passes user input to the AI scheduler.

**Nodes Involved:**  
- When chat message received  
- Window Buffer Memory2  
- Interview Scheduler

**Node Details:**

- **When chat message received**  
  - *Type:* LangChain Chat Trigger  
  - *Role:* Entry point webhook for candidate chat messages, publicly accessible.  
  - *Config:* Public webhook enabled, no additional options.  
  - *Inputs:* External HTTP requests.  
  - *Outputs:* Passes chat input JSON to Interview Scheduler.  
  - *Edge Cases:* Network issues, webhook URL changes, unauthorized access if URL leaked.

- **Window Buffer Memory2**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Maintains conversational context per session using a custom session key (`sessionId`).  
  - *Config:* Context window length set to 10 messages to keep recent conversation history.  
  - *Inputs:* Chat messages from the trigger node.  
  - *Outputs:* Context-enriched input to Interview Scheduler.  
  - *Edge Cases:* Memory overflow if session key misused, session ID collisions.

- **Interview Scheduler**  
  - *Type:* LangChain Agent  
  - *Role:* Core AI agent that processes chat input, asks for interview details (phone, email, preferred time), and confirms before booking.  
  - *Config:*  
    - Model: GPT-4o-mini  
    - System message instructs the AI to:  
      - Use `get_availability` tool to check free slots.  
      - Use `check_days` tool to interpret natural language dates.  
      - Enforce 30-minute interviews, no double bookings, Eastern Time zone.  
      - Output final interview details as JSON only.  
    - Returns intermediate steps for transparency.  
  - *Inputs:* Chat input and memory context.  
  - *Outputs:* JSON or intermediate chat responses.  
  - *Edge Cases:* AI misinterpretation, tool invocation failures, incomplete user input.

---

#### 2.2 AI Processing and Scheduling Logic

**Overview:**  
This block validates AI output, parses structured JSON, and controls the flow depending on whether final interview details are ready.

**Nodes Involved:**  
- If Final Output  
- Convert Output to JSON  
- Respond for More Info

**Node Details:**

- **If Final Output**  
  - *Type:* If Node  
  - *Role:* Checks if AI output contains both `start_datetime` and `end_datetime` indicating a confirmed interview time.  
  - *Config:* String contains check for keys `"start_datetime"` and `"end_datetime"`.  
  - *Inputs:* Output from Interview Scheduler.  
  - *Outputs:*  
    - True branch: proceed to parse JSON.  
    - False branch: trigger no-op node to wait for more info.  
  - *Edge Cases:* Partial or malformed AI output.

- **Convert Output to JSON**  
  - *Type:* LangChain Agent (Output Parser Structured)  
  - *Role:* Parses AI text output into structured JSON format with interview details.  
  - *Config:* JSON schema example provided for validation.  
  - *Inputs:* AI output text.  
  - *Outputs:* Structured JSON for downstream processing.  
  - *Edge Cases:* Parsing errors if AI output deviates from schema.

- **Respond for More Info**  
  - *Type:* No Operation (NoOp)  
  - *Role:* Placeholder node when AI output is incomplete; no action taken.  
  - *Inputs:* False branch from If Final Output.  
  - *Outputs:* None.  
  - *Edge Cases:* None.

---

#### 2.3 Calendar Availability Retrieval and Slot Generation

**Overview:**  
This block retrieves existing calendar events, splits them into 30-minute blocks marked as "Blocked," generates all possible 30-minute weekday slots within business hours, and merges these to identify available times.

**Nodes Involved:**  
- Run Get Availability (Sub-workflow)  
  - Check My Calendar  
  - Split Events into 30 min blocks  
  - Add Blocked Field  
  - Generate 30 Minute Timeslots  
  - Combine My Calendar with All Slots  
  - Check if Calendar Blocked  
  - Return string of all available times  
- Get Availability (Execute Workflow Trigger)

**Node Details:**

- **Get Availability**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point for availability sub-workflow, triggered by Interview Scheduler.  
  - *Config:* Passthrough input source.  
  - *Outputs:* Triggers Check My Calendar and Generate 30 Minute Timeslots nodes in parallel.

- **Check My Calendar**  
  - *Type:* Google Calendar Node  
  - *Role:* Retrieves all events from the specified Google Calendar email.  
  - *Config:*  
    - Operation: getAll  
    - Calendar email: `rbreen.ynteractive@gmail.com` (replace with your own)  
    - Return all events.  
  - *Credentials:* Google Calendar OAuth2.  
  - *Outputs:* List of calendar events.

- **Split Events into 30 min blocks**  
  - *Type:* Code Node  
  - *Role:* Splits each calendar event into 30-minute blocks aligned to half-hour boundaries, formatted in Eastern Time with proper offset.  
  - *Key Logic:*  
    - Aligns event start to previous half-hour slot, end to next half-hour slot.  
    - Creates blocks of 30 minutes each.  
  - *Outputs:* List of blocked time slots.

- **Add Blocked Field**  
  - *Type:* Set Node  
  - *Role:* Adds a `"Blocked"` string field to each 30-minute block to mark it as unavailable.  
  - *Outputs:* Enriched blocked slots.

- **Generate 30 Minute Timeslots**  
  - *Type:* Code Node  
  - *Role:* Generates all possible 30-minute slots on weekdays (Mon-Fri) between 9 AM and 5 PM Eastern Time for the next 7 days, starting 24 hours from now.  
  - *Key Logic:*  
    - Skips weekends.  
    - Rounds current time to next half-hour block.  
    - Formats slots with Eastern Time offset.  
  - *Outputs:* List of potential available slots.

- **Combine My Calendar with All Slots**  
  - *Type:* Merge Node  
  - *Role:* Combines blocked slots with generated slots, enriching the generated slots with any matching blocked flags.  
  - *Config:*  
    - Mode: combine  
    - Join Mode: enrichInput2  
    - Fields to match: `start, end`  
  - *Outputs:* Slots enriched with blocked status where applicable.

- **Check if Calendar Blocked**  
  - *Type:* If Node  
  - *Role:* Filters out slots marked as `"Blocked"`, passing only free slots forward.  
  - *Condition:* `$json.Blocked` not equals `"Blocked"`.  
  - *Outputs:* Available slots only.

- **Return string of all available times**  
  - *Type:* Code Node  
  - *Role:* Formats available slots into a single comma-separated string of `start - end` ranges for AI consumption.  
  - *Outputs:* JSON with `availableSlots` string.

**Edge Cases:**  
- Google API auth failures or rate limits.  
- Time zone misalignment causing incorrect slot generation.  
- Overlapping events not properly split.  
- Empty calendar or no available slots.

---

#### 2.4 Date Interpretation Tool

**Overview:**  
Provides a sub-workflow tool that returns a list of weekday names and dates for the next two weeks to help the AI interpret natural language date references.

**Nodes Involved:**  
- check day names (ToolWorkflow)  
  - When Executed by Another Workflow  
  - Check Day Names (Code Node)  
- Sticky Note1 (documentation)

**Node Details:**

- **check day names**  
  - *Type:* ToolWorkflow Node  
  - *Role:* Sub-workflow that returns weekdays (Mon-Fri) with their dates for the next 14 days.  
  - *Code Node Logic:*  
    - Iterates from today to 14 days ahead.  
    - Filters weekdays only.  
    - Outputs array of strings like `"Monday - 2024-06-03"`.  
  - *Inputs:* Passthrough.  
  - *Outputs:* List of day-date strings.

**Edge Cases:**  
- Time zone differences if server is not in Eastern Time.  
- Date formatting errors.

---

#### 2.5 Booking Confirmation and Calendar Event Creation

**Overview:**  
This block parses the final AI JSON output, creates the Google Calendar event with interview details, and sends a confirmation message back to the candidate.

**Nodes Involved:**  
- Convert Output to JSON  
- Set Meeting with Google  
- Final Response to User  
- If Final Output (from previous block)  
- Sticky Note2 (documentation)

**Node Details:**

- **Convert Output to JSON**  
  - *Type:* LangChain Agent (Output Parser Structured)  
  - *Role:* Parses AI output text into structured JSON with interview info.  
  - *Outputs:* JSON with `interview` object containing email, phone, start and end datetime.

- **Set Meeting with Google**  
  - *Type:* Google Calendar Node  
  - *Role:* Creates a new calendar event for the interview.  
  - *Config:*  
    - Calendar email: `rbreen.ynteractive@gmail.com` (replace with your own)  
    - Start and end times from parsed JSON.  
    - Event summary: "Interview"  
    - Attendee: candidate email.  
    - Description includes candidate phone number.  
  - *Credentials:* Google Calendar OAuth2.  
  - *Outputs:* Confirmation of event creation.

- **Final Response to User**  
  - *Type:* Code Node  
  - *Role:* Constructs a confirmation message with interview details to send back to the candidate.  
  - *Logic:* Extracts email, phone, start, and end datetime from previous node output and formats a user-friendly message.  
  - *Outputs:* Text message.

**Edge Cases:**  
- Google API failures or permission errors.  
- Invalid or missing interview details in JSON.  
- Time zone mismatches causing incorrect event times.

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                              | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                       |
|-------------------------------|--------------------------------------|----------------------------------------------|----------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received     | LangChain Chat Trigger                | Receives candidate chat messages             | -                                | Interview Scheduler               |                                                                                                 |
| Window Buffer Memory2          | LangChain Memory Buffer Window       | Maintains conversational context             | When chat message received        | Interview Scheduler               |                                                                                                 |
| Interview Scheduler            | LangChain Agent                      | AI chatbot guiding interview scheduling      | Window Buffer Memory2             | If Final Output                   |                                                                                                 |
| If Final Output                | If Node                             | Checks if AI output contains final details   | Interview Scheduler              | Convert Output to JSON, Respond for More Info |                                                                                                 |
| Convert Output to JSON         | LangChain Agent (Output Parser)      | Parses AI output to structured JSON           | If Final Output                  | Set Meeting with Google           |                                                                                                 |
| Respond for More Info          | No Operation (NoOp)                  | Placeholder when more info is needed          | If Final Output                  | -                                 |                                                                                                 |
| Set Meeting with Google        | Google Calendar                     | Creates interview event on Google Calendar   | Convert Output to JSON           | Final Response to User            |                                                                                                 |
| Final Response to User         | Code Node                          | Sends confirmation message to candidate      | Set Meeting with Google          | -                                 |                                                                                                 |
| Run Get Availability           | ToolWorkflow (Sub-workflow)          | Retrieves calendar availability               | Interview Scheduler              | Interview Scheduler              | See Sticky Note "Get Availability Execution" for details                                        |
| Check My Calendar              | Google Calendar                     | Retrieves all calendar events                  | Run Get Availability             | Split Events into 30 min blocks   |                                                                                                 |
| Split Events into 30 min blocks| Code Node                         | Splits events into 30-minute blocked slots    | Check My Calendar                | Add Blocked Field                 |                                                                                                 |
| Add Blocked Field              | Set Node                          | Marks slots as blocked                          | Split Events into 30 min blocks  | Combine My Calendar with All Slots|                                                                                                 |
| Generate 30 Minute Timeslots   | Code Node                         | Generates all possible 30-minute slots        | Run Get Availability             | Combine My Calendar with All Slots|                                                                                                 |
| Combine My Calendar with All Slots | Merge Node                    | Merges blocked slots with generated slots     | Add Blocked Field, Generate 30 Minute Timeslots | Check if Calendar Blocked         |                                                                                                 |
| Check if Calendar Blocked      | If Node                          | Filters out blocked slots                       | Combine My Calendar with All Slots| Return string of all available times |                                                                                                 |
| Return string of all available times | Code Node                   | Formats available slots as string for AI       | Check if Calendar Blocked        | Run Get Availability              |                                                                                                 |
| Get Availability              | Execute Workflow Trigger             | Triggers availability sub-workflow             | Interview Scheduler              | Check My Calendar, Generate 30 Minute Timeslots |                                                                                                 |
| check day names               | ToolWorkflow (Sub-workflow)          | Provides weekday names and dates for 2 weeks  | Interview Scheduler              | Interview Scheduler              | See Sticky Note "Check Day Names Tool" for details                                              |
| Sticky Note1                 | Sticky Note                         | Documentation about "Check Day Names Tool"    | -                                | -                                 | See section 2.4                                                                                  |
| Sticky Note2                 | Sticky Note                         | Documentation about "Interview Scheduler" usage| -                                | -                                 | See section 1 overview and deployment steps                                                     |
| Sticky Note                  | Sticky Note                         | Documentation about "Run Get Availability Tool"| -                                | -                                 | See section 2.3                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**
   - Google Calendar OAuth2 credential with your Google account.
   - OpenAI API credential with your API key (GPT-4o access).

2. **Create Webhook Trigger Node:**
   - Add a **LangChain Chat Trigger** node named `When chat message received`.
   - Set it as public.
   - This node receives candidate chat messages.

3. **Add Memory Node:**
   - Add **LangChain Memory Buffer Window** node named `Window Buffer Memory2`.
   - Configure `sessionKey` as `={{ $json.sessionId }}`.
   - Set `contextWindowLength` to 10.
   - Connect `When chat message received` → `Window Buffer Memory2`.

4. **Add Interview Scheduler Agent:**
   - Add **LangChain Agent** node named `Interview Scheduler`.
   - Set model to `gpt-4o-mini`.
   - Paste system message instructing the AI to:
     - Ask for phone, email, preferred date/time.
     - Use `get_availability` tool to check free slots.
     - Use `check_days` tool for natural language date interpretation.
     - Enforce 30-minute interviews, no double bookings.
     - Output final JSON with interview details only.
   - Enable `returnIntermediateSteps`.
   - Connect `Window Buffer Memory2` → `Interview Scheduler`.

5. **Add If Node to Detect Final Output:**
   - Add **If** node named `If Final Output`.
   - Condition: Check if AI output contains `"start_datetime"` AND `"end_datetime"`.
   - Connect `Interview Scheduler` → `If Final Output`.

6. **Add Output Parser Node:**
   - Add **LangChain Agent (Output Parser Structured)** node named `Convert Output to JSON`.
   - Provide JSON schema example for interview details.
   - Connect `If Final Output` (true branch) → `Convert Output to JSON`.

7. **Add NoOp Node:**
   - Add **No Operation** node named `Respond for More Info`.
   - Connect `If Final Output` (false branch) → `Respond for More Info`.

8. **Add Google Calendar Event Creation Node:**
   - Add **Google Calendar** node named `Set Meeting with Google`.
   - Configure calendar email (replace `rbreen.ynteractive@gmail.com` with your own).
   - Set operation to create event with:
     - Start: `={{ $json.output.interview.start_datetime }}`
     - End: `={{ $json.output.interview.end_datetime }}`
     - Summary: "Interview"
     - Attendees: candidate email from JSON
     - Description: candidate phone number
   - Use your Google Calendar OAuth2 credential.
   - Connect `Convert Output to JSON` → `Set Meeting with Google`.

9. **Add Confirmation Message Node:**
   - Add **Code** node named `Final Response to User`.
   - JavaScript to extract email, phone, start, end from previous node and format confirmation text.
   - Connect `Set Meeting with Google` → `Final Response to User`.

10. **Create Availability Sub-workflow:**
    - Create a new workflow named `get_availability`.
    - Add **Execute Workflow Trigger** node named `Get Availability`.
    - Add **Google Calendar** node named `Check My Calendar`:
      - Operation: getAll events.
      - Calendar email: your calendar email.
      - Use Google Calendar OAuth2 credential.
      - Connect `Get Availability` → `Check My Calendar`.
    - Add **Code** node named `Split Events into 30 min blocks`:
      - JavaScript to split events into 30-minute blocks aligned to half-hour slots in Eastern Time.
      - Connect `Check My Calendar` → `Split Events into 30 min blocks`.
    - Add **Set** node named `Add Blocked Field`:
      - Add field `"Blocked"` with value `"Blocked"`.
      - Connect `Split Events into 30 min blocks` → `Add Blocked Field`.
    - Add **Code** node named `Generate 30 Minute Timeslots`:
      - JavaScript to generate all 30-minute slots on weekdays 9 AM–5 PM ET for next 7 days starting 24 hours from now.
      - Connect `Get Availability` → `Generate 30 Minute Timeslots`.
    - Add **Merge** node named `Combine My Calendar with All Slots`:
      - Mode: combine, Join Mode: enrichInput2, match on `start, end`.
      - Connect `Add Blocked Field` → `Combine My Calendar with All Slots` (input 1).
      - Connect `Generate 30 Minute Timeslots` → `Combine My Calendar with All Slots` (input 2).
    - Add **If** node named `Check if Calendar Blocked`:
      - Condition: `$json.Blocked` not equals `"Blocked"`.
      - Connect `Combine My Calendar with All Slots` → `Check if Calendar Blocked`.
    - Add **Code** node named `Return string of all available times`:
      - Formats available slots as comma-separated string `"start - end"`.
      - Connect `Check if Calendar Blocked` → `Return string of all available times`.
    - Set `Return string of all available times` as output of the sub-workflow.

11. **Add ToolWorkflow Node in Main Workflow:**
    - Add **ToolWorkflow** node named `Run Get Availability`.
    - Paste the JSON of the `get_availability` sub-workflow.
    - Connect `Interview Scheduler` → `Run Get Availability`.

12. **Create Date Interpretation Sub-workflow:**
    - Create a new workflow named `check_days`.
    - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.
    - Add **Code** node named `Check Day Names`:
      - JavaScript to generate weekdays (Mon-Fri) with dates for next 14 days.
    - Connect trigger → code node.
    - Set code node output as sub-workflow output.

13. **Add ToolWorkflow Node in Main Workflow:**
    - Add **ToolWorkflow** node named `check day names`.
    - Paste the JSON of the `check_days` sub-workflow.
    - Connect `Interview Scheduler` → `check day names`.

14. **Final Adjustments:**
    - Replace all instances of `rbreen.ynteractive@gmail.com` with your Google Calendar email in all nodes and sub-workflows.
    - Replace credential names `Google Calendar account` and `OpenAi account` with your own credential names.
    - Copy the webhook URL from `When chat message received` node and share it as the public chat interface.
    - Optionally customize system messages, branding, and confirmation text.

15. **Testing and Deployment:**
    - Test Google Calendar and OpenAI API connections.
    - Test chat interaction by sending messages to the webhook URL.
    - Confirm interview scheduling flow completes and events appear on your calendar.

---

This detailed documentation and step-by-step reproduction guide enable advanced users and AI agents to understand, modify, and deploy the Automated Interview Scheduling workflow effectively.