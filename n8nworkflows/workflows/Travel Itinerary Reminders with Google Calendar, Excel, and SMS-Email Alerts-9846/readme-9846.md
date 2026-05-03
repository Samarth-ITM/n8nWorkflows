Travel Itinerary Reminders with Google Calendar, Excel, and SMS/Email Alerts

https://n8nworkflows.xyz/workflows/travel-itinerary-reminders-with-google-calendar--excel--and-sms-email-alerts-9846


# Travel Itinerary Reminders with Google Calendar, Excel, and SMS/Email Alerts

### 1. Workflow Overview

This workflow automates travel itinerary management by daily checking planned trips, syncing them to Google Calendar, and sending personalized reminders via email or SMS to assigned travelers. It targets travel coordinators, agencies, or organizations managing multiple travelers and complex schedules.

The workflow is logically divided into these blocks:

- **1.1 Daily Trigger & Data Retrieval**: Initiates daily execution, reads travel itineraries from Excel.
- **1.2 Trip Filtering & Validation**: Filters trips occurring today or tomorrow evening for reminders, checks if trips exist.
- **1.3 Traveler Data Enrichment**: Reads traveler contact details and matches them to trips.
- **1.4 Reminder Creation & Calendar Sync**: Generates personalized reminder messages, syncs trips to Google Calendar.
- **1.5 Reminder Dispatch Preparation**: Splits reminders into batches, routes based on contact preference (email or SMS), and prepares message content.
- **1.6 Reminder Logging**: Reads existing reminder logs, updates them with new sent reminders, and saves the log.
- **1.7 Auxiliary Nodes**: Wait node to manage timing, sticky notes for documentation.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger & Data Retrieval

**Overview:**  
Triggers the workflow every day and reads the travel itinerary data from an Excel workbook using Microsoft Excel OAuth credentials.

**Nodes Involved:**  
- Daily Travel Check  
- Read Travel Itinerary  

**Node Details:**  
- **Daily Travel Check**  
  - Type: Cron Trigger  
  - Config: Runs daily (default schedule, no custom parameters)  
  - Input: None (trigger node)  
  - Output: Triggers downstream nodes daily  
  - Possible failures: Cron misconfiguration, scheduler disabled

- **Read Travel Itinerary**  
  - Type: Microsoft Excel node (read operation)  
  - Config: Reads travel itinerary Excel file via OAuth2 credentials  
  - Input: Trigger from Daily Travel Check  
  - Output: Returns all travel itinerary rows  
  - Edge cases: Excel file access errors, credential expiry, empty or malformed data  

#### 1.2 Trip Filtering & Validation

**Overview:**  
Filters itinerary data to find trips scheduled for the current day or for tomorrow (if after 6 PM), then checks if any trips exist for processing.

**Nodes Involved:**  
- Filter Today's Trips  
- Has Trips Today?  
- Sync to Google Calendar (parallel path)  

**Node Details:**  
- **Filter Today's Trips**  
  - Type: Code (JavaScript)  
  - Config: Filters trips where the trip date matches today and departure time is within a reminder window; also includes tomorrow's trips if after 6 PM  
  - Key expressions: Uses current date/time, parses trip date and departure time columns, compares reminder hours, combines today’s and tomorrow’s trips  
  - Input: Output from Read Travel Itinerary  
  - Output: Filtered trip list with added metadata (`reminderType`)  
  - Edge cases: Incorrect date/time formats, missing fields, timezone mismatches  

- **Has Trips Today?**  
  - Type: If Node  
  - Config: Checks if the filtered trips include any with a non-empty 'Trip Name' field  
  - Input: Filter Today's Trips output  
  - Output: Routes workflow only if trips exist  
  - Edge cases: Empty trip list, missing Trip Name causing false negatives  

- **Sync to Google Calendar**  
  - Type: Google Calendar node  
  - Config: Creates or updates calendar events for each filtered trip; calculates event start/end time from trip date and departure time, adding 12 hours to end time as a buffer  
  - Credentials: Google OAuth2 required  
  - Input: Filter Today's Trips output (runs in parallel with Has Trips Today?)  
  - Output: Calendar event creation results  
  - Edge cases: API limits, auth token expiry, invalid calendar ID  

#### 1.3 Traveler Data Enrichment

**Overview:**  
Fetches traveler contact information from Excel and associates travelers with trips for personalized reminders.

**Nodes Involved:**  
- Read Traveler Contacts  
- Create Traveler Reminders  

**Node Details:**  
- **Read Traveler Contacts**  
  - Type: Microsoft Excel node (read)  
  - Config: Reads traveler contacts with email, phone, and preferences  
  - Credentials: Microsoft OAuth2  
  - Input: Output from Has Trips Today? (only proceeds if trips exist)  
  - Output: Traveler contact list  
  - Edge cases: Access errors, missing contact info, empty lists  

- **Create Traveler Reminders**  
  - Type: Code (JavaScript)  
  - Config:  
    - Matches travelers to trips based on assigned trips field  
    - For each traveler-trip pair, generates a reminder message text tailored for 'today' or 'tomorrow' reminder type  
    - Includes trip details and travel tips  
  - Key expressions: Uses traveler preferred contact method, trip details, message templates with fallback defaults ('TBD')  
  - Input: Traveler contacts and filtered trips (merged)  
  - Output: Array of reminder messages with contact info and message content  
  - Edge cases: Missing assigned trips, missing contact info, empty arrays  

#### 1.4 Reminder Creation & Calendar Sync (continued)

**Overview:**  
Splits reminders into manageable batches and routes them depending on traveler’s preferred contact method.

**Nodes Involved:**  
- Wait For Data  
- Split Into Batches  
- Email or SMS?  
- Prepare Email Reminders  
- Prepare SMS Reminders  

**Node Details:**  
- **Wait For Data**  
  - Type: Wait  
  - Config: Waits 15 seconds before proceeding (likely to allow data propagation or API rate limits)  
  - Input: Create Traveler Reminders output  
  - Output: Passes data downstream after delay  
  - Edge cases: Workflow timeouts, interrupted wait  

- **Split Into Batches**  
  - Type: SplitInBatches  
  - Config: Batch size set to 10 reminders per batch for processing efficiency  
  - Input: Wait For Data output  
  - Output: Processes reminders in batches downstream  
  - Edge cases: Large datasets may increase total execution time  

- **Email or SMS?**  
  - Type: If Node  
  - Config: Checks if preferred contact method is 'email' to route accordingly  
  - Input: Split Into Batches output  
  - Output: Directs reminders to either email or SMS preparation nodes  
  - Edge cases: Missing or invalid preferred contact values  

- **Prepare Email Reminders**  
  - Type: Code (JavaScript)  
  - Config:  
    - Constructs rich HTML email with styling, trip details, checklist, and personalized greeting  
    - Subject varies depending on reminder type (today/tomorrow)  
    - Includes fallback values for missing trip data  
  - Input: Email branch from Email or SMS? node  
  - Output: Email message payloads including to, subject, body, and html content  
  - Edge cases: HTML rendering issues if data malformed, email address missing  

- **Prepare SMS Reminders**  
  - Type: Code (JavaScript)  
  - Config:  
    - Creates concise SMS text messages tailored to reminder type  
    - Includes essential trip info and preparation notes  
  - Input: SMS branch from Email or SMS? node  
  - Output: SMS message payloads with phone number and message  
  - Edge cases: Phone number missing or invalid, message length limitations  

#### 1.5 Reminder Logging

**Overview:**  
Reads existing reminder logs, updates logs with newly sent reminders (both email and SMS), and appends updated logs back to the Excel file.

**Nodes Involved:**  
- Read Reminder Log  
- Update Reminder Log  
- Save Reminder Log  

**Node Details:**  
- **Read Reminder Log**  
  - Type: Microsoft Excel node (read)  
  - Config: Reads the reminder log Excel file to get existing records  
  - Credentials: Microsoft OAuth2  
  - Input: Output from Prepare Email Reminders and Prepare SMS Reminders (parallel)  
  - Output: Existing logs data  
  - Edge cases: Access or file read errors  

- **Update Reminder Log**  
  - Type: Code (JavaScript)  
  - Config:  
    - Combines existing logs with new entries generated from sent email and SMS reminders  
    - Generates unique log IDs using timestamp and random suffix  
    - Includes metadata such as status ('Sent'), contact method, timestamps, preview of message/subject  
  - Input: Read Reminder Log output + reminders data  
  - Output: Updated log array  
  - Edge cases: Duplicate entries if workflow re-runs rapidly, data consistency issues  

- **Save Reminder Log**  
  - Type: Microsoft Excel node (append operation)  
  - Config: Appends new log entries to reminder log worksheet by workbook and worksheet IDs  
  - Credentials: Microsoft OAuth2  
  - Input: Update Reminder Log output  
  - Output: Confirmation of append operation  
  - Edge cases: Excel API write failures, credential expiry  

#### 1.6 Auxiliary Nodes

**Overview:**  
Sticky notes provide documentation and explanation; no impact on workflow logic.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  

**Node Details:**  
- **Sticky Notes**  
  - Type: Sticky Note nodes (documentation only)  
  - Content: Detailed notes on workflow purpose, prerequisites, data file structures, and component descriptions  
  - Position: Visually grouped for user clarity  
  - No inputs or outputs  

---

### 3. Summary Table

| Node Name               | Node Type                    | Functional Role                                  | Input Node(s)               | Output Node(s)                            | Sticky Note                                                                                                  |
|-------------------------|------------------------------|------------------------------------------------|-----------------------------|------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Daily Travel Check       | Cron Trigger                 | Daily trigger to start workflow                 | None                        | Read Travel Itinerary                    |                                                                                                              |
| Read Travel Itinerary    | Microsoft Excel (read)       | Reads travel itinerary data                      | Daily Travel Check           | Filter Today's Trips                     |                                                                                                              |
| Filter Today's Trips     | Code (JavaScript)            | Filters trips for today and tomorrow reminders  | Read Travel Itinerary        | Has Trips Today?, Sync to Google Calendar |                                                                                                              |
| Has Trips Today?         | If                          | Checks if there are any trips to process         | Filter Today's Trips         | Read Traveler Contacts                   |                                                                                                              |
| Read Traveler Contacts   | Microsoft Excel (read)       | Reads traveler contact info                       | Has Trips Today?             | Create Traveler Reminders                |                                                                                                              |
| Create Traveler Reminders| Code (JavaScript)            | Creates personalized reminders for travelers     | Read Traveler Contacts       | Wait For Data                           |                                                                                                              |
| Wait For Data            | Wait                        | Waits 15 seconds for data stability               | Create Traveler Reminders    | Split Into Batches                       |                                                                                                              |
| Split Into Batches       | SplitInBatches              | Processes reminders in batches                    | Wait For Data                | Email or SMS?                           |                                                                                                              |
| Email or SMS?            | If                          | Routes based on traveler preferred contact       | Split Into Batches           | Prepare Email Reminders, Prepare SMS Reminders |                                                                                                              |
| Prepare Email Reminders  | Code (JavaScript)            | Prepares detailed, styled HTML email reminders   | Email or SMS? (email branch) | Read Reminder Log                       |                                                                                                              |
| Prepare SMS Reminders    | Code (JavaScript)            | Prepares concise SMS reminder messages            | Email or SMS? (SMS branch)   | Read Reminder Log                       |                                                                                                              |
| Sync to Google Calendar  | Google Calendar             | Syncs trips as calendar events                    | Filter Today's Trips         | Create Traveler Reminders                |                                                                                                              |
| Read Reminder Log        | Microsoft Excel (read)       | Reads existing reminder log                       | Prepare Email Reminders, Prepare SMS Reminders | Update Reminder Log             |                                                                                                              |
| Update Reminder Log      | Code (JavaScript)            | Updates reminder log with sent reminders          | Read Reminder Log            | Save Reminder Log                       |                                                                                                              |
| Save Reminder Log        | Microsoft Excel (append)     | Appends updated reminder log to Excel worksheet  | Update Reminder Log          | None                                    |                                                                                                              |
| Sticky Note              | Sticky Note                  | Documentation note                               | None                        | None                                    | ## What This Workflow Does: Automated daily checks, reminders, calendar sync, multi-channel notifications etc.|
| Sticky Note1             | Sticky Note                  | Documentation note                               | None                        | None                                    | ## Essential Prerequisites: Excel files, OAuth, SMTP, SMS provider, etc.                                     |
| Sticky Note2             | Sticky Note                  | Documentation note                               | None                        | None                                    | ## Required Data Files: trip_itinerary.xlsx, traveler_contacts.xlsx, reminder_log.xlsx, and key features      |
| Sticky Note3             | Sticky Note                  | Documentation note                               | None                        | None                                    | ## Main Components: Describes nodes and their roles                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Name: "Daily Travel Check"  
   - Type: Cron Trigger  
   - Schedule: Run once every day (default daily schedule)  

2. **Create Microsoft Excel Node to Read Itinerary**  
   - Name: "Read Travel Itinerary"  
   - Type: Microsoft Excel (read operation)  
   - Credentials: Set up Microsoft Excel OAuth2 credentials  
   - Configure to read the travel itinerary workbook (ensure correct workbook and worksheet selection)  
   - Connect output from "Daily Travel Check"  

3. **Add Code Node to Filter Trips**  
   - Name: "Filter Today's Trips"  
   - Type: Code (JavaScript)  
   - Configure script to:  
     - Get current date and hour  
     - Filter trips where trip date is today and departure time is within reminder window  
     - Include tomorrow’s trips if current time is after 6 PM  
     - Add `reminderType` attribute ('today' or 'tomorrow')  
   - Connect input from "Read Travel Itinerary"  

4. **Add If Node to Check for Trips**  
   - Name: "Has Trips Today?"  
   - Type: If node  
   - Condition: Check if `Trip Name` field is not empty in filtered trips  
   - Connect input from "Filter Today's Trips"  

5. **Add Microsoft Excel Node to Read Traveler Contacts**  
   - Name: "Read Traveler Contacts"  
   - Type: Microsoft Excel (read)  
   - Credentials: Microsoft Excel OAuth2  
   - Configure to read traveler contacts workbook/worksheet  
   - Connect input from "Has Trips Today?" (true branch)  

6. **Add Google Calendar Node to Sync Trips**  
   - Name: "Sync to Google Calendar"  
   - Type: Google Calendar (create/update event)  
   - Credentials: Google OAuth2  
   - Configure calendar ID and event start/end times from trip date and departure time (add 12 hours to end time)  
   - Connect input from "Filter Today's Trips" (parallel to Has Trips Today?)  

7. **Add Code Node to Create Reminders**  
   - Name: "Create Traveler Reminders"  
   - Type: Code (JavaScript)  
   - Logic:  
     - Match travelers to trips based on assigned trips  
     - Generate personalized reminder messages for each traveler-trip pair  
     - Include fallback defaults for missing information  
   - Connect inputs from "Read Traveler Contacts" and "Sync to Google Calendar" outputs (parallel)  

8. **Add Wait Node**  
   - Name: "Wait For Data"  
   - Type: Wait node  
   - Configuration: Wait 15 seconds  
   - Connect input from "Create Traveler Reminders"  

9. **Add SplitInBatches Node**  
   - Name: "Split Into Batches"  
   - Type: SplitInBatches  
   - Batch size: 10  
   - Connect input from "Wait For Data"  

10. **Add If Node to Route by Contact Preference**  
    - Name: "Email or SMS?"  
    - Type: If node  
    - Condition: Check if `preferredContact` equals 'email'  
    - Connect input from "Split Into Batches"  

11. **Add Code Node to Prepare Email Reminders**  
    - Name: "Prepare Email Reminders"  
    - Type: Code (JavaScript)  
    - Logic:  
      - Generate styled HTML email with trip details, checklist, and personalized greeting  
      - Set subject based on reminder type  
    - Connect input from "Email or SMS?" (true branch)  

12. **Add Code Node to Prepare SMS Reminders**  
    - Name: "Prepare SMS Reminders"  
    - Type: Code (JavaScript)  
    - Logic:  
      - Generate concise SMS text messages based on reminder type and trip details  
    - Connect input from "Email or SMS?" (false branch)  

13. **Add Microsoft Excel Node to Read Reminder Log**  
    - Name: "Read Reminder Log"  
    - Type: Microsoft Excel (read)  
    - Credentials: Microsoft Excel OAuth2  
    - Configure to read reminder log workbook  
    - Connect inputs from both "Prepare Email Reminders" and "Prepare SMS Reminders" (parallel)  

14. **Add Code Node to Update Reminder Log**  
    - Name: "Update Reminder Log"  
    - Type: Code (JavaScript)  
    - Logic:  
      - Combine existing log data with new reminder entries for emails and SMS  
      - Generate unique log IDs and timestamps  
    - Connect input from "Read Reminder Log"  

15. **Add Microsoft Excel Node to Save Reminder Log**  
    - Name: "Save Reminder Log"  
    - Type: Microsoft Excel (append)  
    - Credentials: Microsoft Excel OAuth2  
    - Configure to append to reminder log worksheet by workbook and worksheet ID  
    - Connect input from "Update Reminder Log"  

16. **Add Sticky Notes for Documentation (Optional)**  
    - Create sticky notes to document workflow overview, prerequisites, data structure, and main components for clarity  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Automated daily workflow for travel reminders including calendar sync, multi-channel notifications, and logging.                                                             | Sticky Note node content titled "What This Workflow Does"                                                                                                       |
| Requires Microsoft Excel files for itinerary, contacts, and logs; Google Calendar API credentials; SMTP and SMS provider setups.                                              | Sticky Note1 and Sticky Note2 content                                                                                                                           |
| Workflow components include daily trigger, data reading, filtering, contact enrichment, reminders creation, batching, routing, and logging.                                   | Sticky Note3 content                                                                                                                                             |
| Email reminders include styled HTML with pre-travel checklists; SMS reminders are concise and informative.                                                                     | See Prepare Email Reminders and Prepare SMS Reminders node descriptions                                                                                          |
| Reminder logging ensures auditability and prevents duplicate notifications.                                                                                                    | Update Reminder Log and Save Reminder Log nodes                                                                                                                 |
| Google Calendar event end time adds 12 hours to departure time as a buffer, assuming trip duration or activity length may vary.                                               | Sync to Google Calendar node configuration                                                                                                                      |
| Reminder messages include fallback placeholders ('TBD') if any trip detail is missing to maintain message clarity.                                                            | Code in Create Traveler Reminders and Prepare Email Reminders nodes                                                                                            |
| Wait node inserted to manage timing and prevent API rate limits when processing batches.                                                                                        | Wait For Data node                                                                                                                                                |
| Batch processing set to 10 reminders per batch balances throughput and resource usage.                                                                                          | Split Into Batches node configuration                                                                                                                          |
| Preferred contact method defaults to 'email' if not specified.                                                                                                                | Logic in Create Traveler Reminders node                                                                                                                        |

---

This structured documentation provides a complete understanding of the “Automated Journey Scheduler & SMS/Email Alerts” workflow and enables full reproduction, modification, and error anticipation without requiring the raw JSON export.