Automated Voice Appointment Booking with Vapi AI and Google Calendar

https://n8nworkflows.xyz/workflows/automated-voice-appointment-booking-with-vapi-ai-and-google-calendar-8972


# Automated Voice Appointment Booking with Vapi AI and Google Calendar

### 1. Workflow Overview

This workflow automates appointment booking via voice interaction powered by Vapi AI integrated with Google Calendar. It is designed to receive appointment requests from a Vapi AI voice agent, check calendar availability, propose open time slots, and book appointments automatically without manual intervention.

**Target Use Cases:**  
- Businesses using voice AI assistants to schedule appointments (e.g., service providers like roof inspection companies).  
- Automated lead qualification and booking without requiring front-desk staff.  
- Integration of conversational AI with calendar management for seamless scheduling.

**Logical Blocks:**

- **1.1 Input Reception:** Receive webhook calls from Vapi AI containing appointment requests or availability queries.  
- **1.2 Configuration:** Define working hours, time zone, meeting length, and buffer times.  
- **1.3 Routing:** Route incoming requests based on tool call names (`checkAvailability` or `bookAppointment`).  
- **1.4 Availability Checking:**  
  - Retrieve Google Calendar events for the requested day.  
  - Calculate potential appointment slots within working hours.  
  - Filter slots against busy events including buffer times.  
  - Respond with available time slots to Vapi AI.  
- **1.5 Booking:** Create a Google Calendar event for the requested appointment details.  
- **1.6 Confirmation:** Respond to Vapi AI confirming successful booking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives POST requests from Vapi AI voice assistant containing appointment tool calls (either to check availability or book an appointment). It triggers the workflow and passes the incoming data downstream.

- **Nodes Involved:**  
  - `Webhook: Production URL = VAPI Server URL`

- **Node Details:**

  - **Webhook: Production URL = VAPI Server URL**  
    - Type: Webhook  
    - Role: Entry point for Vapi AI calls via HTTP POST.  
    - Configuration:  
      - Path: `AI-Appointment-Setter-Template`  
      - HTTP Method: POST  
      - Response Mode: Response Node (responds from downstream nodes)  
    - Inputs: External HTTP requests from Vapi AI server.  
    - Outputs: JSON with Vapi tool call data (structured in `body.message.toolCalls[0]`).  
    - Edge Cases:  
      - Invalid/malformed requests may cause failure.  
      - Missing or unexpected tool call names will cause routing failures downstream.

#### 2.2 Configuration

- **Overview:**  
  This block holds all configurable parameters defining the appointment setting logic such as timezone, working hours, meeting duration, and buffer times.

- **Nodes Involved:**  
  - `1. CONFIGURATION (EDIT ME)`

- **Node Details:**

  - **1. CONFIGURATION (EDIT ME)**  
    - Type: Set  
    - Role: Stores static configuration parameters for availability and booking.  
    - Configuration:  
      - `timeZone`: e.g., "America/New_York"  
      - `workdayStartHour`: integer, e.g., 9 (9 AM)  
      - `workdayEndHour`: integer, e.g., 17 (5 PM)  
      - `meetingDurationMinutes`: e.g., 30 minutes  
      - `bookingCadenceMinutes`: e.g., 30 (granularity of booking slots)  
      - `bufferBeforeMinutes` & `bufferAfterMinutes`: e.g., 15 minutes buffers before and after meetings  
    - Inputs: Receives webhook JSON but primarily outputs static config.  
    - Outputs: Configuration data used in availability calculation.  
    - Edge Cases:  
      - Misconfigured timezone or hours may yield no slots.  
      - Non-integer or unreasonable values could cause logic errors.

#### 2.3 Routing

- **Overview:**  
  Routes the workflow execution according to the tool call name received from Vapi AI, enabling distinct processing for availability checking or booking.

- **Nodes Involved:**  
  - `Route by Tool Name`

- **Node Details:**

  - **Route by Tool Name**  
    - Type: Switch  
    - Role: Branches workflow based on `message.toolCalls[0].function.name` in webhook JSON.  
    - Configuration:  
      - Two outputs:  
        - `checkAvailability` if function name matches exactly `"checkAvailability"`  
        - `bookAppointment` if function name matches exactly `"bookAppointment"`  
    - Inputs: JSON from `1. CONFIGURATION (EDIT ME)` (which received webhook JSON)  
    - Outputs:  
      - `checkAvailability` path leads to calendar event retrieval.  
      - `bookAppointment` path leads to booking appointment node.  
    - Edge Cases:  
      - Case-sensitive matching requires exact tool call names; misspellings cause no execution.  
      - If tool call name is missing or unexpected, no downstream execution occurs.

#### 2.4 Availability Checking

- **Overview:**  
  This logical block queries Google Calendar for existing events, calculates potential booking slots based on working hours and cadence, filters out busy times (including buffers), and formats available slots for Vapi AI response.

- **Nodes Involved:**  
  - `2. Get Calendar Events (EDIT ME)`  
  - `Calculate Potential Slots (do not change)`  
  - `Filter for Available Slots (do not change)`  
  - `Respond with Available Times (do not change)`

- **Node Details:**

  - **2. Get Calendar Events (EDIT ME)**  
    - Type: Google Calendar  
    - Role: Retrieves all calendar events for the requested date.  
    - Configuration:  
      - Operation: `getAll` events  
      - `timeMin`: initial search datetime from webhook tool call argument  
      - `timeMax`: end of the requested day (calculated)  
      - Calendar: selectable calendar linked via OAuth credentials  
    - Inputs: Routed from `Route by Tool Name` on `checkAvailability` branch  
    - Outputs: Array of calendar events for filtering  
    - Edge Cases:  
      - Credential failures or permission issues may cause errors.  
      - No events found returns empty array (normal).  

  - **Calculate Potential Slots (do not change)**  
    - Type: Code (JavaScript)  
    - Role: Generates all possible booking slots within configured work hours at booking cadence intervals.  
    - Logic:  
      - Parses initial search date from webhook.  
      - Constructs ISO date ranges for workday start and end using config.  
      - Loops to create slots from start to end with cadence intervals.  
    - Inputs:  
      - Config from `1. CONFIGURATION (EDIT ME)`  
      - Initial search datetime from webhook  
    - Outputs: List of potential slots with start and end ISO strings  
    - Edge Cases:  
      - Misconfigured cadence or hours may create zero slots.  
      - Timezone handling relies on correct ISO strings and config.

  - **Filter for Available Slots (do not change)**  
    - Type: Code (JavaScript)  
    - Role: Filters potential slots against existing calendar events plus buffer times to identify truly available slots.  
    - Logic:  
      - Creates intervals from busy events with buffers before/after.  
      - Checks each potential slot for overlap with busy intervals.  
      - Filters out past slots (start before current time).  
      - Formats available slots in human-readable and ISO with timezone offset.  
    - Inputs:  
      - Potential slots from previous node  
      - Calendar events from `2. Get Calendar Events (EDIT ME)`  
      - Config parameters for buffers and meeting duration  
    - Outputs: Array of available slots formatted for Vapi  
    - Edge Cases:  
      - Timezone offset calculation is complex and sensitive to local system time settings.  
      - Overlapping edge cases might arise if event times are irregular.

  - **Respond with Available Times (do not change)**  
    - Type: Respond to Webhook  
    - Role: Sends JSON response back to Vapi AI with available slots for scheduling.  
    - Configuration:  
      - Response body contains toolCallId and serialized availableSlots array.  
    - Inputs: Filtered available slots from previous node.  
    - Outputs: HTTP response to original webhook caller.  
    - Edge Cases:  
      - JSON serialization errors unlikely but possible if unexpected data present.

#### 2.5 Booking Appointment

- **Overview:**  
  This block creates a Google Calendar event for the requested appointment details and confirms the booking to Vapi AI.

- **Nodes Involved:**  
  - `3. Book Appointment in Calendar (EDIT ME)`  
  - `Booking Confirmation (do not change)`

- **Node Details:**

  - **3. Book Appointment in Calendar (EDIT ME)**  
    - Type: Google Calendar  
    - Role: Creates a new calendar event with appointment details from Vapi request.  
    - Configuration:  
      - Start and end times mapped from webhook arguments (`startDateTime`, `endDateTime`).  
      - Calendar selected via OAuth credentials (must be the same as availability check).  
      - Additional fields:  
        - Summary: "Roof Inspection: {clientName}"  
        - Description: Multi-line text with job details, client contact info, and call log ID.  
    - Inputs: Routed from `Route by Tool Name` on `bookAppointment` branch.  
    - Outputs: Confirmation of event creation.  
    - Edge Cases:  
      - Credential or permission failures may cause event creation failure.  
      - Incorrect or missing date/time fields cause errors.  
      - Calendar mismatch between get and create nodes will cause inconsistency.

  - **Booking Confirmation (do not change)**  
    - Type: Respond to Webhook  
    - Role: Sends a JSON confirmation message back to Vapi AI signaling successful booking.  
    - Configuration:  
      - Response contains toolCallId and success message string.  
    - Inputs: From booking node output.  
    - Outputs: HTTP response to Vapi AI.  
    - Edge Cases:  
      - If booking node fails silently, confirmation will be inaccurate (retry enabled on booking node mitigates).

#### 2.6 Supporting Nodes - Sticky Notes

- **Overview:**  
  Provide instructions, tips, and setup guidance for users to configure the workflow properly.

- **Nodes Involved:**  
  - `Sticky Note` and variations (`Sticky Note1`, `Sticky Note2`, `I'm a note`)

- **Node Details:**

  - These nodes contain detailed user guidance on:  
    - How to configure Google Calendar nodes (credentials, calendar selection).  
    - How to set configuration fields (timezone, hours, buffer).  
    - Workflow overview, setup instructions for Vapi, tool naming requirements, and troubleshooting tips.  
    - Suggestions for advanced features and enhancements.  
  - They do not affect workflow execution but are critical for maintainers.

---

### 3. Summary Table

| Node Name                                   | Node Type                 | Functional Role                         | Input Node(s)                           | Output Node(s)                                       | Sticky Note                                                                                                                         |
|---------------------------------------------|---------------------------|---------------------------------------|---------------------------------------|-----------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Webhook: Production URL = VAPI Server URL   | Webhook                   | Entry point for Vapi AI tool calls    | (External HTTP)                       | 1. CONFIGURATION (EDIT ME)                          |                                                                                                                                    |
| 1. CONFIGURATION (EDIT ME)                   | Set                       | Holds configuration parameters        | Webhook: Production URL                | Route by Tool Name                                  | Sticky Note1: Edit Fields: timezone, work hours, meeting duration, buffers                                                        |
| Route by Tool Name                           | Switch                    | Routes workflow by tool call name     | 1. CONFIGURATION (EDIT ME)             | 2. Get Calendar Events (checkAvailability branch), 3. Book Appointment in Calendar (bookAppointment branch) |                                                                                                                                    |
| 2. Get Calendar Events (EDIT ME)             | Google Calendar           | Retrieves calendar events for the day | Route by Tool Name (checkAvailability)| Calculate Potential Slots (do not change)            | Sticky Note: Edit Google Calendar nodes: connect credentials and select calendar                                                  |
| Calculate Potential Slots (do not change)    | Code                      | Generates all potential time slots    | 2. Get Calendar Events (EDIT ME)       | Filter for Available Slots (do not change)           |                                                                                                                                    |
| Filter for Available Slots (do not change)   | Code                      | Filters slots against busy events     | Calculate Potential Slots              | Respond with Available Times (do not change)         |                                                                                                                                    |
| Respond with Available Times (do not change) | Respond to Webhook        | Returns available slots to Vapi AI    | Filter for Available Slots             | (HTTP response to webhook caller)                    |                                                                                                                                    |
| 3. Book Appointment in Calendar (EDIT ME)   | Google Calendar           | Creates calendar event for booking    | Route by Tool Name (bookAppointment)  | Booking Confirmation (do not change)                  | Sticky Note: Edit Google Calendar nodes: connect credentials and select calendar                                                  |
| Booking Confirmation (do not change)          | Respond to Webhook        | Confirms booking to Vapi AI            | 3. Book Appointment in Calendar        | (HTTP response to webhook caller)                    |                                                                                                                                    |
| Sticky Note                                  | Sticky Note               | Setup instructions and guidance       | None                                  | None                                                | See content for Google Calendar setup instructions                                                                                |
| Sticky Note1                                 | Sticky Note               | Configuration field editing tips      | None                                  | None                                                | Explains timezone, work hours, meeting duration, buffer setup                                                                    |
| Sticky Note2                                 | Sticky Note               | Full workflow overview and setup tips | None                                  | None                                                | Detailed workflow description, setup instructions, troubleshooting, security notes                                                |
| I'm a note                                   | Sticky Note               | Suggestions for advanced features     | None                                  | None                                                | Suggestions for SMS, email confirmations, CRM integration, multilingual support, call transferring, etc.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Webhook: Production URL = VAPI Server URL`  
   - HTTP Method: POST  
   - Path: `AI-Appointment-Setter-Template`  
   - Response Mode: `responseNode` (to respond downstream)  

2. **Create Set Node for Configuration**  
   - Type: Set  
   - Name: `1. CONFIGURATION (EDIT ME)`  
   - Parameters (assignments):  
     - `timeZone`: `"America/New_York"` (example)  
     - `workdayStartHour`: 9  
     - `workdayEndHour`: 17  
     - `meetingDurationMinutes`: 30  
     - `bookingCadenceMinutes`: 30  
     - `bufferBeforeMinutes`: 15  
     - `bufferAfterMinutes`: 15  
   - Connect output from Webhook node.

3. **Create Switch Node for Routing**  
   - Type: Switch  
   - Name: `Route by Tool Name`  
   - Rules:  
     - Output 1: If expression equals `"checkAvailability"` for `{{$json.body.message.toolCalls[0].function.name}}`  
     - Output 2: If expression equals `"bookAppointment"` for same field  
   - Connect output from Configuration node.

4. **Create Google Calendar Node to Get Events**  
   - Type: Google Calendar  
   - Name: `2. Get Calendar Events (EDIT ME)`  
   - Operation: Get All Events  
   - TimeMin: `={{ $json.body.message.toolCalls[0].function.arguments.initialSearchDateTime }}`  
   - TimeMax: `={{ DateTime.fromISO($json.body.message.toolCalls[0].function.arguments.initialSearchDateTime).endOf('day').toISO() }}`  
   - Calendar: Select your calendar with Google OAuth credential  
   - Connect from Switch Node output for `checkAvailability`.

5. **Create Code Node to Calculate Potential Slots**  
   - Type: Code (JavaScript)  
   - Name: `Calculate Potential Slots (do not change)`  
   - Code: Implement logic to generate slots within work hours at booking cadence intervals based on config and initial search date.  
   - Connect from Get Calendar Events node.

6. **Create Code Node to Filter Available Slots**  
   - Type: Code (JavaScript)  
   - Name: `Filter for Available Slots (do not change)`  
   - Code: Implement logic to exclude busy intervals plus buffers and past times, format available slots for response.  
   - Connect from Calculate Potential Slots node.

7. **Create Respond to Webhook Node to Return Slots**  
   - Type: Respond to Webhook  
   - Name: `Respond with Available Times (do not change)`  
   - Response Body: JSON containing `toolCallId` and serialized `availableSlots`  
   - Connect from Filter for Available Slots node.

8. **Create Google Calendar Node to Book Appointment**  
   - Type: Google Calendar  
   - Name: `3. Book Appointment in Calendar (EDIT ME)`  
   - Operation: Create Event  
   - Start: `={{ $json.body.message.toolCalls[0].function.arguments.startDateTime }}`  
   - End: `={{ $json.body.message.toolCalls[0].function.arguments.endDateTime }}`  
   - Calendar: Use same calendar and OAuth credential as Get Events node  
   - Summary: `"Roof Inspection: {{$json.body.message.toolCalls[0].function.arguments.clientName}}"`  
   - Description: Multiline text with service type, address, client contact info, call log ID (use expressions to map fields from webhook JSON)  
   - Retry on Fail: Enabled  
   - Connect from Switch Node output for `bookAppointment`.

9. **Create Respond to Webhook Node to Confirm Booking**  
   - Type: Respond to Webhook  
   - Name: `Booking Confirmation (do not change)`  
   - Response Body: JSON with `toolCallId` and `"The appointment has been successfully booked."` message  
   - Connect from Book Appointment node.

10. **Connect Workflow**  
    - Webhook output → Configuration node  
    - Configuration output → Switch node  
    - Switch `checkAvailability` output → Get Calendar Events → Calculate Slots → Filter Slots → Respond Available Times  
    - Switch `bookAppointment` output → Book Appointment → Booking Confirmation

11. **Credential Setup**  
    - Create Google OAuth credential in n8n with Calendar scope.  
    - Assign this credential to both Google Calendar nodes for consistency.

12. **Vapi Setup** (outside n8n)  
    - Configure Vapi Assistant to send toolCalls with exact names: `checkAvailability` and `bookAppointment`.  
    - Set webhook URL in Vapi to this workflow’s webhook URL.  
    - Ensure tool call arguments match expected fields (dates, clientName, serviceType, etc.).

13. **Activate Workflow**  
    - Turn workflow active in n8n.  
    - Test end-to-end by initiating calls or sessions via Vapi AI.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                         |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This template connects a Vapi AI voice agent to Google Calendar for automated appointment booking without code.                 | Sticky Note2 content                                   |
| Ensure Google OAuth credentials have proper calendar scopes and consent for event reading and creation.                          | Google Calendar node notes                             |
| Tool call names in Vapi must exactly match `checkAvailability` and `bookAppointment` or routing will fail silently.             | Sticky Note2 troubleshooting section                   |
| Suggested enhancements: SMS/email confirmations, multilingual support, CRM integration, call transferring, and rescheduling.  | Sticky Note "I'm a note"                               |
| Workflow uses n8n version supporting webhook response nodes and advanced JavaScript code nodes (v2+ recommended).              | Implicit from node versions                             |
| For setup, consult https://streetlamp.agency for expert support and further resources.                                          | Sticky Note "I'm a note"                               |
| No API keys are hardcoded; all credentials managed securely in n8n.                                                             | Sticky Note2 security notes                             |

---

_Disclaimer: The content provided here is derived solely from an automated workflow created with n8n, respecting all current content policies. No illegal, offensive, or protected data is included. All data handled is legal and public._